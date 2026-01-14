# Order Service

## Description

The Order Service is TechCorp's central order processing microservice. This component orchestrates the entire lifecycle of an order, from creation to completion, coordinating interactions with inventory-service, payment-service, and notification-service.

The service implements a robust state machine to manage order status transitions (PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED). Each transition triggers events that are consumed by other services to maintain system consistency.

To ensure resilience, the order-service uses the Saga pattern for distributed transactions, allowing automatic compensation in case of failures at any stage of the order process.

## Owners

- **Team:** Commerce Engineering
- **Tech Lead:** Juliana Almeida
- **Slack:** #commerce-orders

## Technology Stack

- Language: Java 21
- Framework: Spring Boot 3.2 + Spring State Machine
- Database: PostgreSQL 15
- Cache: Redis 7
- Message Broker: RabbitMQ 3.12
- Search: Elasticsearch 8.11

## Configuration

### Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `ORDER_SERVICE_PORT` | Service HTTP port | `3003` |
| `DATABASE_URL` | PostgreSQL connection string | - |
| `REDIS_URL` | Redis URL | - |
| `RABBITMQ_URL` | RabbitMQ URL | - |
| `ELASTICSEARCH_URL` | Elasticsearch URL | - |
| `INVENTORY_SERVICE_URL` | inventory-service URL | - |
| `PAYMENT_SERVICE_URL` | payment-service URL | - |
| `NOTIFICATION_SERVICE_URL` | notification-service URL | - |
| `SAGA_TIMEOUT_SECONDS` | Timeout for saga compensation | `300` |

### State Machine Configuration

```yaml
# application.yml
order:
  state-machine:
    states:
      - PENDING
      - PAYMENT_PENDING
      - PAYMENT_CONFIRMED
      - PROCESSING
      - SHIPPED
      - DELIVERED
      - CANCELLED
      - REFUNDED
    transitions:
      - from: PENDING
        to: PAYMENT_PENDING
        event: CONFIRM
      - from: PAYMENT_PENDING
        to: PAYMENT_CONFIRMED
        event: PAYMENT_RECEIVED
      - from: PAYMENT_CONFIRMED
        to: PROCESSING
        event: START_PROCESSING
```

## How to Run Locally

```bash
# Clone the repository
git clone git@github.com:techcorp/order-service.git
cd order-service

# Start dependencies
docker-compose up -d postgres redis rabbitmq elasticsearch

# Compile
./mvnw clean package -DskipTests

# Run migrations
./mvnw flyway:migrate

# Start the service
./mvnw spring-boot:run -Dspring.profiles.active=local

# Verify health
curl http://localhost:3003/actuator/health
```

### Create Test Order

```bash
curl -X POST http://localhost:3003/api/orders \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "user_id": "uuid",
    "items": [
      {"product_id": "uuid", "quantity": 2}
    ],
    "shipping_address_id": "uuid"
  }'
```

## Order State Machine

```
                    +-----------+
                    |  PENDING  |
                    +-----+-----+
                          |
                    CONFIRM
                          |
                    +-----v-------+
                    | PAYMENT     |
                    | PENDING     |
                    +-----+-------+
                          |
              +-----------+-----------+
              |                       |
        PAYMENT_RECEIVED         PAYMENT_FAILED
              |                       |
        +-----v-------+         +-----v-----+
        | PAYMENT     |         | CANCELLED |
        | CONFIRMED   |         +-----------+
        +-----+-------+
              |
        START_PROCESSING
              |
        +-----v-------+
        | PROCESSING  |
        +-----+-------+
              |
           SHIP
              |
        +-----v-----+
        |  SHIPPED  |
        +-----+-----+
              |
          DELIVER
              |
        +-----v------+
        | DELIVERED  |
        +------------+
```

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/order-service
- **Prometheus Metrics:** https://prometheus.techcorp.internal/targets
- **Logs:** Sent to Elasticsearch via Fluentd

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `orders_created_total` | Total orders created | - |
| `orders_completed_total` | Completed orders | - |
| `orders_cancelled_total` | Cancelled orders | > 5% |
| `order_processing_duration_seconds` | Processing time | > 30s |
| `saga_compensation_total` | Saga compensations | > 1/min |

### Configured Alerts

- **OrderServiceHighCancellationRate:** Cancellation rate above 5%
- **OrderSagaCompensationHigh:** More than 10 compensations per minute
- **OrderProcessingTimeout:** Orders stuck in PROCESSING for more than 1 hour

## Troubleshooting

### Issue: Order stuck in PAYMENT_PENDING status

**Cause:** Timeout in communication with payment-service or webhook not received.

**Solution:**
1. Check status in payment-service: `GET /payments?order_id=<id>`
2. If payment confirmed, manually trigger event: `POST /admin/orders/<id>/events/payment_received`
3. If payment doesn't exist, cancel order: `POST /admin/orders/<id>/cancel`

### Issue: Saga executing unexpected compensation

**Cause:** Saga timeout configured too low or downstream service slow.

**Solution:**
1. Check saga logs: filter by `saga.order.<order_id>`
2. Identify which step failed in saga history
3. Increase timeout if necessary or investigate slow service

### Issue: Slow order search

**Cause:** Elasticsearch index outdated or poorly optimized query.

**Solution:**
1. Force reindex: `POST /admin/orders/reindex`
2. Check index mapping
3. Analyze query in Kibana Dev Tools

## Consumed Events

| Event | Source | Action |
|-------|--------|--------|
| `payment.confirmed` | payment-service | Transitions to PAYMENT_CONFIRMED |
| `payment.failed` | payment-service | Transitions to CANCELLED |
| `inventory.reserved` | inventory-service | Marks items as reserved |
| `shipping.delivered` | logistics | Transitions to DELIVERED |

## Published Events

| Event | Exchange | Description |
|-------|----------|-------------|
| `order.created` | `order.events` | New order created |
| `order.confirmed` | `order.events` | Order confirmed |
| `order.shipped` | `order.events` | Order shipped |
| `order.delivered` | `order.events` | Order delivered |
| `order.cancelled` | `order.events` | Order cancelled |

## Related Links

- [Orders API](../apis/orders-api.md) - Available REST resources
- [Payment Service](payment-service.md) - Payment processing
- [Inventory Service](inventory-service.md) - Inventory control
- [Notification Service](notification-service.md) - Status notifications
- [Data Flow](../architecture/data-flow.md) - Complete order flow
