# Erros Comuns e Soluções

## Visão Geral

Este documento lista os erros mais frequentes encontrados na plataforma TechCorp e suas soluções. Use esta referência para diagnóstico rápido de problemas.

## Erros de Autenticação

### Erro: "Token expirado" (401)

**Mensagem:**
```json
{
  "error": "token_expired",
  "message": "O token de acesso expirou"
}
```

**Causa:** Access token JWT expirou (validade de 30 minutos).

**Solução:**
1. Usar refresh token para obter novo access token
2. Se refresh token também expirou, solicitar novo login

**Código de exemplo:**
```javascript
if (error.code === 'token_expired') {
  const newTokens = await authService.refresh(refreshToken);
  // Retry original request with new token
}
```

---

### Erro: "Credenciais inválidas" (401)

**Mensagem:**
```json
{
  "error": "invalid_credentials",
  "message": "E-mail ou senha incorretos"
}
```

**Causa:** E-mail não cadastrado ou senha incorreta.

**Solução:**
1. Verificar se e-mail está correto
2. Usar fluxo de recuperação de senha
3. Verificar se conta não está bloqueada

**Investigação (admin):**
```sql
SELECT email, status, failed_login_attempts, locked_until
FROM auth.users_credentials
WHERE email = 'user@example.com';
```

---

### Erro: "Conta bloqueada" (403)

**Mensagem:**
```json
{
  "error": "account_locked",
  "message": "Conta bloqueada após múltiplas tentativas"
}
```

**Causa:** 5 tentativas de login falhas consecutivas.

**Solução:**
1. Aguardar 30 minutos (desbloqueio automático)
2. Usar recuperação de senha
3. Contatar suporte para desbloqueio manual

**Desbloqueio manual (admin):**
```sql
UPDATE auth.users_credentials
SET locked_until = NULL, failed_login_attempts = 0
WHERE email = 'user@example.com';
```

---

### Erro: "MFA requerido" (401)

**Mensagem:**
```json
{
  "error": "mfa_required",
  "mfa_token": "temporary_token",
  "message": "Autenticação de dois fatores necessária"
}
```

**Causa:** Usuário tem MFA habilitado, precisa enviar código.

**Solução:**
1. Solicitar código do app authenticator ou SMS
2. Enviar código para `/auth/mfa/verify` com `mfa_token`

---

## Erros de Pedidos

### Erro: "Produto sem estoque" (409)

**Mensagem:**
```json
{
  "error": "out_of_stock",
  "message": "Produto indisponível",
  "product_id": "uuid"
}
```

**Causa:** Estoque zerou entre adicionar ao carrinho e finalizar checkout.

**Solução para usuário:**
1. Remover item do carrinho
2. Verificar alternativas (outras cores/tamanhos)

**Investigação (admin):**
```sql
SELECT product_id, available, reserved
FROM inventory.stock_levels
WHERE product_id = 'uuid';
```

---

### Erro: "Pedido não pode ser cancelado" (400)

**Mensagem:**
```json
{
  "error": "invalid_operation",
  "message": "Pedido já foi enviado e não pode ser cancelado"
}
```

**Causa:** Pedido está em status SHIPPED ou posterior.

**Solução:**
1. Aguardar entrega e solicitar devolução
2. Recusar entrega (se possível)
3. Contatar suporte para casos especiais

---

### Erro: "Cupom inválido" (422)

**Mensagem:**
```json
{
  "error": "invalid_coupon",
  "message": "Cupom expirado ou não aplicável"
}
```

**Causas possíveis:**
- Cupom expirou
- Valor mínimo não atingido
- Cupom já utilizado (uso único)
- Cupom não aplicável aos itens do carrinho

**Verificação (admin):**
```sql
SELECT code, expires_at, min_order_value, usage_limit, used_count
FROM coupons.coupons
WHERE code = 'DESCONTO10';
```

---

## Erros de Pagamento

### Erro: "Pagamento recusado" (402)

**Mensagem:**
```json
{
  "error": "payment_declined",
  "decline_code": "insufficient_funds",
  "message": "Pagamento recusado pelo emissor"
}
```

**Códigos de recusa comuns:**

| Código | Causa | Ação |
|--------|-------|------|
| `insufficient_funds` | Saldo insuficiente | Usar outro cartão |
| `card_declined` | Cartão bloqueado | Contatar banco |
| `expired_card` | Cartão vencido | Usar cartão válido |
| `invalid_cvv` | CVV incorreto | Verificar código |
| `fraud_suspected` | Suspeita de fraude | Contatar banco |

**Investigação (admin):**
```sql
SELECT transaction_id, status, decline_code, provider_response
FROM payments.transactions
WHERE order_id = 'uuid';
```

---

### Erro: "Token de cartão expirado" (422)

**Mensagem:**
```json
{
  "error": "token_expired",
  "message": "Token do cartão expirou"
}
```

**Causa:** Token do cartão (gerado no frontend) tem validade de 15 minutos.

**Solução:**
1. Solicitar ao usuário que insira os dados do cartão novamente
2. Verificar se há delay excessivo no checkout

---

### Erro: "PIX expirado" (422)

**Mensagem:**
```json
{
  "error": "pix_expired",
  "message": "Código PIX expirou"
}
```

**Causa:** PIX tem validade padrão de 30 minutos.

**Solução:**
1. Gerar novo código PIX
2. Recomeçar checkout se pedido foi cancelado

---

## Erros de API

### Erro: "Rate limit excedido" (429)

**Mensagem:**
```json
{
  "error": "rate_limit_exceeded",
  "message": "Muitas requisições",
  "retry_after": 60
}
```

**Causa:** Cliente excedeu limite de requisições.

**Limites padrão:**

| Tipo | Limite |
|------|--------|
| API pública | 100 req/min |
| API autenticada | 1000 req/min |
| Login | 5 req/min |

**Solução:**
1. Aguardar tempo indicado em `retry_after`
2. Implementar backoff exponencial
3. Otimizar chamadas (caching, batching)

---

### Erro: "Serviço indisponível" (503)

**Mensagem:**
```json
{
  "error": "service_unavailable",
  "message": "Serviço temporariamente indisponível"
}
```

**Causa:** Serviço downstream não está respondendo.

**Solução para usuário:**
1. Aguardar alguns minutos e tentar novamente
2. Verificar status page: status.techcorp.com

**Investigação (admin):**
```bash
# Verificar status dos pods
kubectl get pods -n production

# Verificar logs recentes
kubectl logs -n production -l app=<service> --tail=100
```

---

### Erro: "Bad Gateway" (502)

**Mensagem:**
```json
{
  "error": "bad_gateway",
  "message": "Erro de comunicação com serviço"
}
```

**Causa:** API Gateway não conseguiu se comunicar com serviço upstream.

**Investigação:**
1. Verificar se serviço está rodando
2. Verificar network policies
3. Verificar se há circuit breaker aberto

---

### Erro: "Gateway Timeout" (504)

**Mensagem:**
```json
{
  "error": "gateway_timeout",
  "message": "Tempo limite excedido"
}
```

**Causa:** Serviço upstream demorou mais de 30 segundos para responder.

**Investigação:**
1. Verificar latência do serviço no Grafana
2. Verificar queries lentas no banco
3. Verificar se há pico de tráfego

---

## Erros de Busca

### Erro: "Busca retornando zero resultados"

**Causa possível:** Query muito restritiva ou índice desatualizado.

**Investigação:**
```bash
# Verificar se termo existe no índice
curl -X GET "elasticsearch:9200/products/_search?q=<termo>"

# Verificar status do índice
curl -X GET "elasticsearch:9200/_cat/indices/products"
```

**Solução:**
1. Ampliar filtros de busca
2. Verificar sinônimos configurados
3. Forçar reindexação se necessário

---

## Erros de Upload

### Erro: "Arquivo muito grande" (413)

**Mensagem:**
```json
{
  "error": "file_too_large",
  "message": "Arquivo excede o limite de 5MB"
}
```

**Limites:**

| Tipo | Limite |
|------|--------|
| Avatar | 5MB |
| Imagem de review | 10MB |
| CSV de importação | 50MB |

**Solução:**
1. Comprimir imagem antes de enviar
2. Redimensionar para tamanho menor

---

### Erro: "Tipo de arquivo não permitido" (415)

**Mensagem:**
```json
{
  "error": "unsupported_media_type",
  "message": "Tipo de arquivo não suportado"
}
```

**Tipos permitidos:**

| Contexto | Tipos |
|----------|-------|
| Avatar | JPG, PNG |
| Review | JPG, PNG, GIF |
| Importação | CSV |

## Links Relacionados

- [Problemas de Performance](performance-issues.md) - Otimização
- [FAQ de Integrações](integration-faq.md) - Integrações
- [API de Autenticação](../apis/auth-api.md) - Endpoints de auth
- [API de Pagamentos](../apis/payments-api.md) - Operações de pagamento
