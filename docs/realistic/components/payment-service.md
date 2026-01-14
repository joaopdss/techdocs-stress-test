# Payment Service

## Description

The Payment Service is the microservice responsible for processing all financial transactions on the TechCorp platform. This component integrates with multiple payment providers (gateways, acquirers, and digital wallets) to offer various payment options to customers.

The service implements an abstraction layer over different providers, allowing new payment methods to be added without changing consumer services. It also manages the complete lifecycle of transactions, including authorization, capture, cancellation, and refunds.

To ensure transaction security, the payment-service is PCI-DSS Level 1 certified and does not store sensitive card data. All information is tokenized directly at the payment providers.

## Owners

- **Team:** Financial Engineering
- **Tech Lead:** Bruno Carvalho
- **Slack:** #fineng-payments

## Technology Stack

- Language: Go 1.21
- Framework: Gin + Wire (DI)
- Database: PostgreSQL 15
- Cache: Redis 7
- Message Broker: RabbitMQ 3.12

## Integrated Providers

| Provider | Methods | Status |
|----------|---------|--------|
| Stripe | Credit card, debit | Active |
| PagSeguro | Card, boleto, PIX | Active |
| MercadoPago | Card, boleto, PIX | Active |
| PayPal | PayPal Wallet | Active |
| Apple Pay | Apple Wallet | Beta |

## Configuration

### Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `PAYMENT_SERVICE_PORT` | Service HTTP port | `3004` |
| `DATABASE_URL` | PostgreSQL connection string | - |
| `REDIS_URL` | Redis URL | - |
| `RABBITMQ_URL` | RabbitMQ URL | - |
| `STRIPE_API_KEY` | Stripe API key | - |
| `STRIPE_WEBHOOK_SECRET` | Secret for validating Stripe webhooks | - |
| `PAGSEGURO_TOKEN` | PagSeguro token | - |
| `MERCADOPAGO_ACCESS_TOKEN` | MercadoPago token | - |
| `ENCRYPTION_KEY` | Key for sensitive data encryption | - |

### Provider Configuration

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

## How to Run Locally

```bash
# Clone the repository
git clone git@github.com:techcorp/payment-service.git
cd payment-service

# Start dependencies
docker-compose up -d postgres redis rabbitmq

# Configure sandbox variables
cp .env.example .env.local
# Edit .env.local with provider sandbox tokens

# Compile
go build -o payment-service ./cmd/server

# Run migrations
./payment-service migrate

# Start the service
./payment-service serve

# Verify health
curl http://localhost:3004/health
```

### Simulate Sandbox Payment

```bash
# Create payment with Stripe test card
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

## Payment Flow

```
Client -> Payment Service -> Provider Gateway -> Issuing Bank
                |                    |
                v                    v
         Save transaction      Process payment
                |                    |
                v                    v
         Publish event    <-- Response (approved/denied)
```

### Transaction States

| State | Description |
|-------|-------------|
| `PENDING` | Transaction created, awaiting processing |
| `AUTHORIZED` | Payment authorized, not captured |
| `CAPTURED` | Payment captured successfully |
| `FAILED` | Payment declined |
| `CANCELLED` | Transaction cancelled before capture |
| `REFUNDED` | Refund processed |
| `PARTIALLY_REFUNDED` | Partial refund processed |

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/payment-service
- **Prometheus Metrics:** https://prometheus.techcorp.internal/targets
- **Logs:** Sent to Elasticsearch via Fluentd

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `payments_total` | Total transactions | - |
| `payments_amount_total` | Financial volume processed | - |
| `payments_success_rate` | Approval rate | < 95% |
| `payments_latency_seconds` | Processing latency | > 5s |
| `payments_by_provider` | Transactions by provider | - |

### Configured Alerts

- **PaymentSuccessRateLow:** Approval rate below 95% for 10 minutes
- **PaymentProviderDown:** Provider not responding for 2 minutes
- **PaymentHighLatency:** P99 latency above 10 seconds
- **PaymentRefundRateHigh:** Refund rate above 2%

## Troubleshooting

### Issue: Payment returning invalid card error

**Cause:** Card token expired or invalid.

**Solution:**
1. Check if token hasn't expired (Stripe tokens expire in 24h)
2. Request customer to enter data again
3. Check provider logs for error details

### Issue: Provider webhook not being received

**Cause:** Incorrect callback URL or firewall blocking.

**Solution:**
1. Check URL configured in provider dashboard
2. Test connectivity: webhook should arrive at `/webhooks/<provider>`
3. Check API Gateway logs for blocked requests

### Issue: Refund not processed

**Cause:** Original transaction not found or refund period expired.

**Solution:**
1. Check original transaction ID: `GET /api/payments/<id>`
2. Confirm it's within refund period (usually 180 days)
3. Check available balance in provider account

### Issue: Duplicate charge

**Cause:** Automatic retry after timeout without idempotency key.

**Solution:**
1. Identify duplicate transactions by `order_id`
2. Keep only the first approved transaction
3. Process refund for others automatically

## Consumed Events

| Event | Source | Action |
|-------|--------|--------|
| `order.created` | order-service | Prepare to receive payment |
| `order.cancelled` | order-service | Cancel/refund payment |

## Published Events

| Event | Exchange | Description |
|-------|----------|-------------|
| `payment.authorized` | `payment.events` | Payment authorized |
| `payment.captured` | `payment.events` | Payment captured |
| `payment.failed` | `payment.events` | Payment declined |
| `payment.refunded` | `payment.events` | Refund processed |

## Related Links

- [Payments API](../apis/payments-api.md) - Available operations
- [Order Service](order-service.md) - Order processing
- [Security Model](../architecture/security-model.md) - PCI-DSS compliance
- [Incident Response](../runbooks/incident-response.md) - Emergency procedures
