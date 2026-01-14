# Payment Service

## Descrição

O Payment Service é o microsserviço responsável por processar todas as transações financeiras da plataforma TechCorp. Este componente integra-se com múltiplos provedores de pagamento (gateways, adquirentes e wallets digitais) para oferecer diversas opções de pagamento aos clientes.

O serviço implementa uma camada de abstração sobre os diferentes provedores, permitindo adicionar novos métodos de pagamento sem alterar os serviços consumidores. Ele também gerencia o ciclo de vida completo das transações, incluindo autorização, captura, cancelamento e estornos.

Para garantir a segurança das transações, o payment-service é certificado PCI-DSS Level 1 e não armazena dados sensíveis de cartão. Todas as informações são tokenizadas diretamente nos provedores de pagamento.

## Responsáveis

- **Time:** Financial Engineering
- **Tech Lead:** Bruno Carvalho
- **Slack:** #fineng-payments

## Stack Tecnológica

- Linguagem: Go 1.21
- Framework: Gin + Wire (DI)
- Banco de dados: PostgreSQL 15
- Cache: Redis 7
- Message Broker: RabbitMQ 3.12

## Provedores Integrados

| Provedor | Métodos | Status |
|----------|---------|--------|
| Stripe | Cartão de crédito, débito | Ativo |
| PagSeguro | Cartão, boleto, PIX | Ativo |
| MercadoPago | Cartão, boleto, PIX | Ativo |
| PayPal | PayPal Wallet | Ativo |
| Apple Pay | Apple Wallet | Beta |

## Configuração

### Variáveis de Ambiente

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| `PAYMENT_SERVICE_PORT` | Porta HTTP do serviço | `3004` |
| `DATABASE_URL` | Connection string PostgreSQL | - |
| `REDIS_URL` | URL do Redis | - |
| `RABBITMQ_URL` | URL do RabbitMQ | - |
| `STRIPE_API_KEY` | Chave da API Stripe | - |
| `STRIPE_WEBHOOK_SECRET` | Secret para validar webhooks Stripe | - |
| `PAGSEGURO_TOKEN` | Token PagSeguro | - |
| `MERCADOPAGO_ACCESS_TOKEN` | Token MercadoPago | - |
| `ENCRYPTION_KEY` | Chave para criptografia de dados sensíveis | - |

### Configuração de Provedores

```yaml
# config/providers.yaml
providers:
  stripe:
    enabled: true
    sandbox: false
    api_version: "2023-10-16"
    webhook_tolerance: 300
  pagseguro:
    enabled: true
    sandbox: false
    notification_url: https://api.techcorp.com/webhooks/pagseguro
  mercadopago:
    enabled: true
    sandbox: false
```

## Como Executar Localmente

```bash
# Clonar o repositório
git clone git@github.com:techcorp/payment-service.git
cd payment-service

# Subir dependências
docker-compose up -d postgres redis rabbitmq

# Configurar variáveis de sandbox
cp .env.example .env.local
# Editar .env.local com tokens de sandbox dos provedores

# Compilar
go build -o payment-service ./cmd/server

# Executar migrações
./payment-service migrate

# Iniciar o serviço
./payment-service serve

# Verificar saúde
curl http://localhost:3004/health
```

### Simular Pagamento em Sandbox

```bash
# Criar pagamento com cartão de teste Stripe
curl -X POST http://localhost:3004/api/payments \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "order_id": "uuid",
    "amount": 10000,
    "currency": "BRL",
    "method": "credit_card",
    "provider": "stripe",
    "card_token": "tok_visa"
  }'
```

## Fluxo de Pagamento

```
Cliente -> Payment Service -> Provider Gateway -> Banco Emissor
                |                    |
                v                    v
         Salvar transação      Processar pagamento
                |                    |
                v                    v
         Publicar evento    <-- Resposta (aprovado/negado)
```

### Estados da Transação

| Estado | Descrição |
|--------|-----------|
| `PENDING` | Transação criada, aguardando processamento |
| `AUTHORIZED` | Pagamento autorizado, não capturado |
| `CAPTURED` | Pagamento capturado com sucesso |
| `FAILED` | Pagamento recusado |
| `CANCELLED` | Transação cancelada antes da captura |
| `REFUNDED` | Estorno realizado |
| `PARTIALLY_REFUNDED` | Estorno parcial realizado |

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/payment-service
- **Métricas Prometheus:** https://prometheus.techcorp.internal/targets
- **Logs:** Enviados para Elasticsearch via Fluentd

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `payments_total` | Total de transações | - |
| `payments_amount_total` | Volume financeiro processado | - |
| `payments_success_rate` | Taxa de aprovação | < 95% |
| `payments_latency_seconds` | Latência do processamento | > 5s |
| `payments_by_provider` | Transações por provedor | - |

### Alertas Configurados

- **PaymentSuccessRateLow:** Taxa de aprovação abaixo de 95% por 10 minutos
- **PaymentProviderDown:** Provedor não responde por 2 minutos
- **PaymentHighLatency:** Latência P99 acima de 10 segundos
- **PaymentRefundRateHigh:** Taxa de estorno acima de 2%

## Troubleshooting

### Problema: Pagamento retornando erro de cartão inválido

**Causa:** Token do cartão expirado ou inválido.

**Solução:**
1. Verificar se o token não expirou (tokens Stripe expiram em 24h)
2. Solicitar ao cliente que insira os dados novamente
3. Verificar logs do provedor para detalhes do erro

### Problema: Webhook do provedor não está sendo recebido

**Causa:** URL de callback incorreta ou firewall bloqueando.

**Solução:**
1. Verificar URL configurada no dashboard do provedor
2. Testar conectividade: webhook deve chegar em `/webhooks/<provider>`
3. Verificar logs do API Gateway para requisições bloqueadas

### Problema: Estorno não processado

**Causa:** Transação original não encontrada ou período de estorno expirado.

**Solução:**
1. Verificar ID da transação original: `GET /api/payments/<id>`
2. Confirmar que está dentro do período de estorno (geralmente 180 dias)
3. Verificar saldo disponível na conta do provedor

### Problema: Duplicidade de cobrança

**Causa:** Retry automático após timeout sem idempotency key.

**Solução:**
1. Identificar transações duplicadas pelo `order_id`
2. Manter apenas a primeira transação aprovada
3. Processar estorno das demais automaticamente

## Eventos Consumidos

| Evento | Origem | Ação |
|--------|--------|------|
| `order.created` | order-service | Preparar para receber pagamento |
| `order.cancelled` | order-service | Cancelar/estornar pagamento |

## Eventos Publicados

| Evento | Exchange | Descrição |
|--------|----------|-----------|
| `payment.authorized` | `payment.events` | Pagamento autorizado |
| `payment.captured` | `payment.events` | Pagamento capturado |
| `payment.failed` | `payment.events` | Pagamento recusado |
| `payment.refunded` | `payment.events` | Estorno processado |

## Links Relacionados

- [API de Pagamentos](../apis/payments-api.md) - Endpoints disponíveis
- [Order Service](order-service.md) - Processamento de pedidos
- [Modelo de Segurança](../architecture/security-model.md) - Conformidade PCI-DSS
- [Resposta a Incidentes](../runbooks/incident-response.md) - Procedimentos de emergência
