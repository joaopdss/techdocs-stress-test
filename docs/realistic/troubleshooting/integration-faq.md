# FAQ de Integrações

## Visão Geral

Este documento responde às perguntas mais frequentes sobre integrações com a plataforma TechCorp, incluindo APIs, webhooks e sistemas externos.

## Autenticação e Autorização

### Como obtenho credenciais de API?

Para obter credenciais de API para integração:

1. Acesse o Portal de Desenvolvedores: developers.techcorp.com
2. Crie uma conta ou faça login
3. Registre uma nova aplicação
4. Obtenha seu `client_id` e `client_secret`

**Importante:** O `client_secret` é exibido apenas uma vez. Armazene-o de forma segura.

---

### Qual a diferença entre client_credentials e authorization_code?

| Fluxo | Uso | Quando usar |
|-------|-----|-------------|
| `client_credentials` | Server-to-server | Integração backend sem usuário |
| `authorization_code` | User delegation | App agindo em nome do usuário |

**Exemplo client_credentials:**
```bash
curl -X POST https://api.techcorp.com/v1/auth/oauth/token \
  -d grant_type=client_credentials \
  -d client_id=<your_client_id> \
  -d client_secret=<your_client_secret>
```

---

### Meu token expirou, o que faço?

Access tokens expiram em 30 minutos. Use o refresh token para obter um novo:

```bash
curl -X POST https://api.techcorp.com/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "<your_refresh_token>"}'
```

Se o refresh token também expirou (7 dias), você precisa autenticar novamente.

---

### Como testo a API em ambiente de sandbox?

Use o ambiente de sandbox para testes:

- **Base URL:** `https://api.sandbox.techcorp.com/v1`
- **Portal:** `https://sandbox.techcorp.com`

Credenciais de sandbox são separadas de produção. Crie uma conta no portal de sandbox.

---

## Webhooks

### Como configuro webhooks?

1. Acesse o Portal de Desenvolvedores
2. Vá em Configurações > Webhooks
3. Adicione a URL do seu endpoint
4. Selecione os eventos que deseja receber
5. Salve e copie o `webhook_secret`

---

### Quais eventos de webhook estão disponíveis?

| Evento | Descrição |
|--------|-----------|
| `order.created` | Novo pedido criado |
| `order.paid` | Pagamento confirmado |
| `order.shipped` | Pedido enviado |
| `order.delivered` | Pedido entregue |
| `order.cancelled` | Pedido cancelado |
| `payment.confirmed` | Pagamento aprovado |
| `payment.failed` | Pagamento recusado |
| `payment.refunded` | Estorno processado |
| `user.created` | Novo usuário registrado |
| `user.updated` | Dados de usuário alterados |

---

### Como valido a assinatura do webhook?

Todos os webhooks incluem o header `X-Webhook-Signature` com assinatura HMAC-SHA256:

```python
import hmac
import hashlib

def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)

# Uso
is_valid = verify_webhook(
    request.body,
    request.headers['X-Webhook-Signature'],
    WEBHOOK_SECRET
)
```

---

### Webhook não está chegando, o que verifico?

Checklist de troubleshooting:

1. **URL acessível?** Endpoint deve ser público (HTTPS)
2. **Resposta rápida?** Responda com 2xx em menos de 5 segundos
3. **Certificado válido?** SSL/TLS deve estar válido
4. **Firewall?** Whitelist nossos IPs (veja abaixo)
5. **Evento configurado?** Verifique se o evento está selecionado

**IPs para whitelist:**
```
54.123.45.67
54.123.45.68
54.123.45.69
```

---

### O que acontece se meu endpoint estiver fora do ar?

Implementamos retry automático com backoff exponencial:

| Tentativa | Delay |
|-----------|-------|
| 1 | Imediato |
| 2 | 1 minuto |
| 3 | 5 minutos |
| 4 | 30 minutos |
| 5 | 2 horas |

Após 5 falhas, o webhook é movido para a DLQ. Você pode reprocessar via Portal.

---

## Pagamentos

### Quais provedores de pagamento são suportados?

| Provedor | Métodos | Status |
|----------|---------|--------|
| Stripe | Cartão crédito/débito | Ativo |
| PagSeguro | Cartão, Boleto, PIX | Ativo |
| MercadoPago | Cartão, Boleto, PIX | Ativo |
| PayPal | PayPal Wallet | Ativo |

---

### Como processo pagamentos na minha integração?

Para processar pagamentos via API:

1. **Tokenize o cartão** no frontend usando nosso SDK
2. **Envie o token** para seu backend
3. **Chame a API de pagamentos** com o token

```javascript
// Frontend: Tokenizar cartão
const token = await TechCorpPay.createToken({
  number: '4242424242424242',
  exp_month: 12,
  exp_year: 2025,
  cvv: '123'
});

// Backend: Processar pagamento
const payment = await fetch('/api/payments/process', {
  method: 'POST',
  body: JSON.stringify({
    order_id: 'uuid',
    amount: 29990,
    card_token: token.id
  })
});
```

---

### Como testo pagamentos em sandbox?

Use os cartões de teste:

| Número | Resultado |
|--------|-----------|
| 4242424242424242 | Aprovado |
| 4000000000000002 | Recusado |
| 4000000000009995 | Saldo insuficiente |
| 4000000000000069 | Cartão expirado |

Para PIX de teste, use o código QR retornado - o pagamento é confirmado automaticamente em 5 segundos.

---

## Dados e Sincronização

### Como sincronizo produtos com meu sistema?

Opções de sincronização:

1. **Pull via API:** Chame `GET /products` periodicamente
2. **Push via Webhook:** Receba eventos `product.updated`
3. **Feed completo:** Baixe CSV diário do Portal

Recomendamos webhook para atualizações em tempo real + feed diário para reconciliação.

---

### Qual o rate limit da API?

| Tipo | Limite | Janela |
|------|--------|--------|
| API autenticada | 1000 req | 1 minuto |
| API pública | 100 req | 1 minuto |
| Endpoints de busca | 60 req | 1 minuto |
| Endpoints de escrita | 30 req | 1 minuto |

Headers de resposta indicam uso:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1705312260
```

---

### Como faço paginação corretamente?

Use paginação por cursor para grandes volumes:

```bash
# Primeira página
GET /api/orders?limit=100

# Resposta inclui cursor
{
  "data": [...],
  "next_cursor": "eyJpZCI6IjEyMyJ9"
}

# Próxima página
GET /api/orders?limit=100&cursor=eyJpZCI6IjEyMyJ9
```

**Não use** offset para grandes volumes - performance degrada.

---

## Erros e Debugging

### Como debugo erros de integração?

1. **Verifique o código de erro:** Consulte [Erros Comuns](common-errors.md)
2. **Verifique headers de resposta:** `X-Request-Id` para suporte
3. **Verifique logs no Portal:** Histórico de chamadas por 7 dias
4. **Contate suporte:** Inclua `X-Request-Id`

---

### Erro 429: O que fazer?

Você excedeu o rate limit. Soluções:

1. **Implemente backoff exponencial:**
```python
import time

def call_with_retry(func, max_retries=5):
    for i in range(max_retries):
        try:
            return func()
        except RateLimitError as e:
            wait = 2 ** i
            time.sleep(wait)
    raise Exception("Max retries exceeded")
```

2. **Otimize chamadas:**
   - Use caching local
   - Batch operations quando possível
   - Evite polling desnecessário

3. **Solicite aumento de limite:** Contate suporte com justificativa

---

### Erro de certificado SSL, o que fazer?

Se estiver recebendo erros de SSL:

1. **Verifique data/hora:** Seu servidor deve ter horário correto
2. **Atualize CA certificates:** `apt-get update && apt-get install ca-certificates`
3. **Verifique proxy:** Proxies corporativos podem interferir

**Nunca** desabilite verificação de certificado em produção.

---

## Suporte

### Como contato o suporte técnico?

| Canal | Uso | SLA |
|-------|-----|-----|
| Portal de Desenvolvedores | Dúvidas gerais | 48h |
| developers@techcorp.com | Issues técnicos | 24h |
| Slack #api-support | Parceiros Enterprise | 4h |

Sempre inclua:
- `X-Request-Id` do erro
- Timestamp do problema
- Request/Response (sem dados sensíveis)

---

### Onde encontro a documentação completa?

- **API Reference:** docs.techcorp.com/api
- **Guias:** docs.techcorp.com/guides
- **SDKs:** github.com/techcorp/sdks
- **Status:** status.techcorp.com

## Links Relacionados

- [Erros Comuns](common-errors.md) - Códigos de erro
- [API de Autenticação](../apis/auth-api.md) - Endpoints de auth
- [API de Pedidos](../apis/orders-api.md) - Recursos REST de pedidos
- [API de Pagamentos](../apis/payments-api.md) - Operações de pagamento
