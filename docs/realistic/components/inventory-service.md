# Inventory Service

## Description

The Inventory Service is the microservice responsible for TechCorp's inventory control. This component manages product availability in real-time, performing reservations, stock deductions, and replenishments across multiple warehouses and distribution centers.

The service implements an inventory model with temporal reservations, where items are reserved during the checkout process and automatically released if payment is not confirmed within a configurable period. This prevents overselling while maintaining availability for other customers.

For high-volume operations, the inventory-service uses distributed counters with Redis to ensure performance and eventual consistency. Periodic synchronizations reconcile counters with the main database.

## Owners

- **Team:** Supply Chain Tech
- **Tech Lead:** Marcos Ribeiro
- **Slack:** #supply-inventory

## Technology Stack

- Language: Go 1.21
- Framework: Chi Router
- Database: PostgreSQL 15
- Cache: Redis 7 (distributed counters)
- Message Broker: RabbitMQ 3.12

## Configuration

### Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `INVENTORY_PORT` | Service HTTP port | `3006` |
| `DATABASE_URL` | PostgreSQL connection string | - |
| `REDIS_URL` | Redis URL | - |
| `RABBITMQ_URL` | RabbitMQ URL | - |
| `RESERVATION_TTL_MINUTES` | Reservation time to live | `15` |
| `SYNC_INTERVAL_SECONDS` | Redis->DB sync interval | `60` |
| `LOW_STOCK_THRESHOLD` | Low stock alert threshold | `10` |

### Warehouse Configuration

```yaml
# config/warehouses.yaml
warehouses:
  - id: wh-sp-001
    name: DC Sao Paulo
    region: southeast
    priority: 1
  - id: wh-rj-001
    name: DC Rio de Janeiro
    region: southeast
    priority: 2
  - id: wh-pr-001
    name: DC Curitiba
    region: south
    priority: 1
```

## How to Run Locally

```bash
# Clone the repository
git clone git@github.com:techcorp/inventory-service.git
cd inventory-service

# Start dependencies
docker-compose up -d postgres redis rabbitmq

# Compile
go build -o inventory-service ./cmd/server

# Run migrations
./inventory-service migrate

# Seed test data
./inventory-service seed --warehouses --products

# Start the service
./inventory-service serve

# Verify health
curl http://localhost:3006/health
```

### Check Inventory

```bash
# Check product availability
curl http://localhost:3006/api/inventory/products/uuid/availability

# Reserve inventory
curl -X POST http://localhost:3006/api/inventory/reservations \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": "uuid",
    "items": [
      {"product_id": "uuid", "quantity": 2, "warehouse_id": "wh-sp-001"}
    ]
  }'
```

## Data Model

### StockLevel Entity

```go
type StockLevel struct {
    ProductID    uuid.UUID
    WarehouseID  string
    Available    int       // Quantity available for sale
    Reserved     int       // Quantity reserved (checkout in progress)
    Incoming     int       // Quantity in transit for replenishment
    UpdatedAt    time.Time
}
```

### Reservation Entity

```go
type Reservation struct {
    ID           uuid.UUID
    OrderID      uuid.UUID
    Items        []ReservationItem
    Status       string    // ACTIVE, CONFIRMED, EXPIRED, CANCELLED
    ExpiresAt    time.Time
    CreatedAt    time.Time
}
```

## Inventory Operations

### Reservation Flow

```
1. Checkout starts -> Create reservation with 15 min TTL
2. Payment confirmed -> Convert reservation to stock deduction
3. Payment fails/timeout -> Reservation expires, stock released
```

### Allocation Strategy

The service automatically selects the warehouse based on:

1. **Availability:** Warehouse must have sufficient stock
2. **Proximity:** Prioritizes warehouse closest to delivery ZIP code
3. **Priority:** Region priority configuration
4. **Balancing:** Distributes load across warehouses

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/inventory-service
- **Prometheus Metrics:** https://prometheus.techcorp.internal/targets
- **Logs:** Sent to Elasticsearch via Fluentd

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `inventory_reservations_total` | Total reservations | - |
| `inventory_reservations_expired` | Expired reservations | > 10% |
| `inventory_stock_level` | Stock level by product | < threshold |
| `inventory_sync_duration_seconds` | Sync duration | > 10s |
| `inventory_oversell_events` | Oversell events | > 0 |

### Configured Alerts

- **InventoryLowStock:** Product with stock below threshold
- **InventoryOversell:** Sale above available stock detected
- **InventorySyncFailed:** Redis->DB sync failed
- **InventoryReservationExpiredHigh:** Expiration rate above 10%

## Troubleshooting

### Issue: Stock showing available but order failing

**Cause:** Inconsistency between Redis counter and database.

**Solution:**
1. Force sync: `POST /admin/inventory/sync`
2. Check values: `GET /admin/inventory/products/<id>/debug`
3. If necessary, reset counter: `POST /admin/inventory/products/<id>/reset-counter`

### Issue: Reservations not expiring

**Cause:** Expiration worker stopped or Redis issue.

**Solution:**
1. Check worker: `kubectl logs -l app=inventory-expiration-worker`
2. Check TTL in Redis: `redis-cli TTL reservation:<id>`
3. Expire manually if urgent: `POST /admin/reservations/<id>/expire`

### Issue: Negative stock detected

**Cause:** Race condition in concurrent operations or counter bug.

**Solution:**
1. Identify affected product in logs
2. Temporarily block sales: `POST /admin/products/<id>/block-sales`
3. Manually reconcile stock with physical inventory
4. Unblock after correction

### Issue: Slow synchronization

**Cause:** High product volume or slow database connection.

**Solution:**
1. Check PostgreSQL latency metrics
2. Increase sync parallelism: `SYNC_WORKERS=10`
3. Consider partial sync by warehouse

## Consumed Events

| Event | Source | Action |
|-------|--------|--------|
| `order.created` | order-service | Create stock reservation |
| `payment.confirmed` | payment-service | Confirm reservation (deduct) |
| `payment.failed` | payment-service | Cancel reservation |
| `order.cancelled` | order-service | Return stock |

## Published Events

| Event | Exchange | Description |
|-------|----------|-------------|
| `inventory.reserved` | `inventory.events` | Stock reserved |
| `inventory.confirmed` | `inventory.events` | Reservation confirmed |
| `inventory.released` | `inventory.events` | Stock released |
| `inventory.low_stock` | `inventory.events` | Low stock alert |
| `inventory.out_of_stock` | `inventory.events` | Product out of stock |

## Related Links

- [Order Service](order-service.md) - Order processing
- [Products API](../apis/products-api.md) - Product API paths
- [Data Flow](../architecture/data-flow.md) - Inventory integration in the flow
- [Performance Issues](../troubleshooting/performance-issues.md) - Optimizations
