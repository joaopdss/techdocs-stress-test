# Data Flow

## Overview

This document describes how data flows through TechCorp systems, from user interaction to final persistence. Understanding these flows is essential for problem diagnosis and performance optimization.

## Complete Order Flow

### Sequence Diagram

```
Client    Gateway    Order    Inventory   Payment    Notification
   │          │         │          │          │            │
   ├─────────►│         │          │          │            │
   │  POST    │         │          │          │            │
   │ /orders  ├────────►│          │          │            │
   │          │ Create  ├─────────►│          │            │
   │          │ order   │ Reserve  │          │            │
   │          │         │ stock    │          │            │
   │          │         │◄─────────┤          │            │
   │          │         │   OK     │          │            │
   │          │         ├──────────┼─────────►│            │
   │          │         │          │ Process  │            │
   │          │         │          │ payment  │            │
   │          │         │◄─────────┼──────────┤            │
   │          │         │          │   OK     │            │
   │          │         ├─────────►│          │            │
   │          │         │ Confirm  │          │            │
   │          │         │ reserv.  │          │            │
   │          │         ├──────────┼──────────┼───────────►│
   │          │         │          │          │   Send     │
   │          │         │          │          │   email    │
   │◄─────────┤◄────────┤          │          │            │
   │   200    │  Order  │          │          │            │
   │          │ created │          │          │            │
```

### Step Details

#### 1. Order Creation

**Origin:** Web Portal / Mobile App
**Destination:** Order Service via API Gateway

```json
{
  "cart_id": "uuid",
  "shipping_address_id": "uuid",
  "payment": {
    "method": "credit_card",
    "card_token": "tok_xxx"
  }
}
```

**Transformations:**
1. API Gateway validates JWT
2. API Gateway adds context headers
3. Order Service validates input data
4. Order Service creates record in `orders.orders`

#### 2. Stock Reservation

**Origin:** Order Service
**Destination:** Inventory Service

```json
{
  "order_id": "uuid",
  "items": [
    {"product_id": "uuid", "quantity": 2}
  ],
  "reservation_ttl_minutes": 15
}
```

**Transformations:**
1. Inventory Service checks availability
2. Inventory Service creates reservation with TTL
3. Inventory Service updates Redis counters
4. Inventory Service records in `inventory.reservations`

**Persisted Data:**
- Table: `inventory.reservations`
- Redis: `inventory:product:{id}:available` (decrement)

#### 3. Payment Processing

**Origin:** Order Service
**Destination:** Payment Service

```json
{
  "order_id": "uuid",
  "amount": 29990,
  "currency": "BRL",
  "method": "credit_card",
  "card_token": "tok_xxx"
}
```

**Transformations:**
1. Payment Service creates transaction (status: PENDING)
2. Payment Service sends to external provider
3. Provider returns authorization
4. Payment Service updates transaction (status: APPROVED)
5. Payment Service publishes `payment.confirmed` event

**Persisted Data:**
- Table: `payments.transactions`
- Event: `payment.confirmed` on RabbitMQ

#### 4. Reservation Confirmation

**Origin:** Order Service (via event)
**Destination:** Inventory Service

```json
{
  "event": "payment.confirmed",
  "order_id": "uuid"
}
```

**Transformations:**
1. Inventory Service receives event
2. Inventory Service converts reservation to stock deduction
3. Inventory Service updates actual stock

**Persisted Data:**
- Table: `inventory.stock_movements`
- `inventory.reservations` updated (status: CONFIRMED)

#### 5. Customer Notification

**Origin:** Order Service (via event)
**Destination:** Notification Service

```json
{
  "event": "order.confirmed",
  "user_id": "uuid",
  "order_id": "uuid",
  "order_number": "TC-2024-001234"
}
```

**Transformations:**
1. Notification Service receives event
2. Notification Service loads template
3. Notification Service renders with order data
4. Notification Service sends via provider (SendGrid)

**Persisted Data:**
- Table: `notifications.delivery_log`

## Authentication Flow

```
Client    Gateway    Auth      Redis     PostgreSQL
   │          │         │         │          │
   ├─────────►│         │         │          │
   │  POST    │         │         │          │
   │ /login   ├────────►│         │          │
   │          │         ├─────────┼─────────►│
   │          │         │         │  Fetch   │
   │          │         │         │  user    │
   │          │         │◄────────┼──────────┤
   │          │         │         │          │
   │          │         │ Validate│          │
   │          │         │ password│          │
   │          │         │         │          │
   │          │         ├────────►│          │
   │          │         │ Create  │          │
   │          │         │ session │          │
   │          │         │◄────────┤          │
   │          │         │         │          │
   │          │         │ Generate│          │
   │          │         │ JWT     │          │
   │◄─────────┤◄────────┤         │          │
   │   JWT    │         │         │          │
```

### JWT Token Data

```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user_uuid",
    "email": "user@example.com",
    "iat": 1705312200,
    "exp": 1705314000,
    "iss": "https://auth.techcorp.com",
    "aud": "techcorp-services"
  }
}
```

### Session Storage

```
Redis Key: session:{session_id}
TTL: 7 days

Value: {
  "user_id": "uuid",
  "refresh_token": "xxx",
  "device_info": {...},
  "created_at": "2024-01-15T10:00:00Z"
}
```

## Product Search Flow

```
Client    Gateway    Search    Redis    Elasticsearch
   │          │         │         │          │
   ├─────────►│         │         │          │
   │  GET     │         │         │          │
   │ /search  ├────────►│         │          │
   │ ?q=xxx   │         ├────────►│          │
   │          │         │  Cache  │          │
   │          │         │  lookup │          │
   │          │         │◄────────┤          │
   │          │         │ MISS    │          │
   │          │         ├─────────┼─────────►│
   │          │         │         │  Query   │
   │          │         │◄────────┼──────────┤
   │          │         │         │          │
   │          │         ├────────►│          │
   │          │         │ Cache   │          │
   │          │         │ store   │          │
   │◄─────────┤◄────────┤         │          │
   │ Results  │         │         │          │
```

### Elasticsearch Query

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "blue t-shirt",
            "fields": ["title^2", "description"]
          }
        }
      ],
      "filter": [
        {"term": {"in_stock": true}},
        {"range": {"price": {"gte": 50, "lte": 200}}}
      ]
    }
  },
  "sort": [
    {"_score": "desc"},
    {"popularity_score": "desc"}
  ]
}
```

## Indexing Flow

```
Catalog    RabbitMQ    Search    Elasticsearch
Service
   │          │          │            │
   │ Product  │          │            │
   │ updated  │          │            │
   ├─────────►│          │            │
   │ product. │          │            │
   │ updated  │          │            │
   │          ├─────────►│            │
   │          │ Consume  │            │
   │          │          ├───────────►│
   │          │          │  Index     │
   │          │          │  document  │
   │          │          │◄───────────┤
   │          │          │    ACK     │
   │          │◄─────────┤            │
   │          │   ACK    │            │
```

### Update Event

```json
{
  "event": "product.updated",
  "product_id": "uuid",
  "changes": ["price", "stock"],
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### Indexed Document

```json
{
  "id": "uuid",
  "title": "Basic Cotton T-Shirt",
  "description": "100% cotton t-shirt...",
  "price": 89.90,
  "original_price": 119.90,
  "category": "t-shirts",
  "brand": "techcorp-basics",
  "attributes": {
    "color": ["blue", "red", "black"],
    "size": ["s", "m", "l", "xl"]
  },
  "in_stock": true,
  "popularity_score": 0.85,
  "updated_at": "2024-01-15T10:30:00Z"
}
```

## Data Consistency

### Saga Pattern

For operations involving multiple services, we use the Saga pattern with compensation:

```
Order Service orchestrates:

1. Reserve Inventory
   ├── Success → Continue
   └── Failure → Abort

2. Process Payment
   ├── Success → Continue
   └── Failure → Release Inventory

3. Confirm Reservation
   ├── Success → Complete
   └── Failure → Refund Payment, Release Inventory
```

### Event Sourcing

Critical events are stored for auditing and replay:

```
orders.order_events:
- order_id: uuid
- event_type: CREATED | PAID | SHIPPED | DELIVERED
- event_data: {...}
- created_at: timestamp
```

## Data Replication

### PostgreSQL

```
Primary (R/W) ──► Replica 1 (R/O)
                  Replica 2 (R/O)
                  Analytics Replica (R/O)
```

### Redis

```
Cluster Mode:
Master 1 ──► Slave 1
Master 2 ──► Slave 2
Master 3 ──► Slave 3
```

## Related Links

- [Architecture Overview](system-overview.md) - System architecture
- [Integration Patterns](integration-patterns.md) - Messaging patterns
- [Order Service](../components/order-service.md) - Order service
- [Inventory Service](../components/inventory-service.md) - Inventory service
