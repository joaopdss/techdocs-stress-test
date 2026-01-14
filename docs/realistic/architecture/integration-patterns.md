# Integration Patterns

## Overview

This document describes the integration patterns used for communication between TechCorp microservices. Choosing the appropriate pattern depends on the consistency, latency, and resilience requirements of each use case.

## Synchronous Patterns

### Request-Response via REST

**When to use:**
- Operations that require immediate response
- Simple data queries
- Real-time validations

**Example: Validate stock availability**

```
Order Service                 Inventory Service
     │                              │
     │  GET /inventory/{id}/check   │
     ├─────────────────────────────►│
     │                              │
     │     { available: true }      │
     │◄─────────────────────────────┤
```

**Implementation:**

```python
# Order Service
async def check_inventory(product_id: str, quantity: int) -> bool:
    response = await http_client.get(
        f"{INVENTORY_URL}/products/{product_id}/availability",
        params={"quantity": quantity},
        timeout=5.0
    )
    return response.json()["available"]
```

### Circuit Breaker

**When to use:**
- Protect against cascade failures
- Services with history of instability
- Critical operations with fallback

**States:**

```
CLOSED ──────────► OPEN ──────────► HALF-OPEN
   │                 │                  │
   │ N consecutive   │ Timeout          │ 1 success
   │ failures        │ expired          │
   │                 │                  │
   └─────────────────┴──────────────────┘
```

**Implementation:**

```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30)
async def process_payment(order_id: str, amount: int):
    return await payment_client.charge(order_id, amount)

# Fallback when circuit is open
async def process_payment_with_fallback(order_id: str, amount: int):
    try:
        return await process_payment(order_id, amount)
    except CircuitBreakerError:
        # Queue for later processing
        await queue.publish("payments.retry", {"order_id": order_id, "amount": amount})
        return {"status": "queued"}
```

### Retry with Exponential Backoff

**When to use:**
- Transient network failures
- Occasional timeouts
- Temporary rate limiting

**Strategy:**

```
Attempt 1: immediate
Attempt 2: after 1s
Attempt 3: after 2s
Attempt 4: after 4s
Attempt 5: after 8s (maximum)
```

**Implementation:**

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=1, max=8)
)
async def call_external_service(payload):
    return await http_client.post(url, json=payload)
```

## Asynchronous Patterns

### Publish-Subscribe (Fanout)

**When to use:**
- Multiple consumers interested in the same event
- Decoupling between producer and consumers
- Domain events

**Example: Order created event**

```
Order Service
     │
     │  order.created
     ▼
┌────────────────────┐
│   Exchange        │
│   (fanout)        │
└────┬────┬────┬────┘
     │    │    │
     ▼    ▼    ▼
  Queue Queue Queue
    │     │     │
    ▼     ▼     ▼
Inventory  Notification  Analytics
Service    Service       Service
```

**Implementation:**

```python
# Producer (Order Service)
async def create_order(order_data):
    order = await order_repository.create(order_data)

    await message_broker.publish(
        exchange="events.fanout",
        routing_key="",
        message={
            "event": "order.created",
            "order_id": order.id,
            "user_id": order.user_id,
            "items": order.items,
            "timestamp": datetime.utcnow().isoformat()
        }
    )

    return order

# Consumer (Notification Service)
@consumer(queue="notification.order-events")
async def handle_order_created(message):
    if message["event"] == "order.created":
        await send_order_confirmation_email(
            user_id=message["user_id"],
            order_id=message["order_id"]
        )
```

### Topic-Based Routing

**When to use:**
- Consumers interested in subset of events
- Pattern-based routing
- Event filtering at broker

**Example: Routing by notification type**

```
Notification Service
     │
     │  notification.email.order.confirmed
     │  notification.sms.order.shipped
     │  notification.push.promo.new
     ▼
┌────────────────────┐
│   Exchange        │
│   (topic)         │
└────┬────┬────┬────┘
     │    │    │
     ▼    ▼    ▼
notification. notification. notification.
email.*       sms.*         push.*
     │         │              │
     ▼         ▼              ▼
Email       SMS Worker    Push Worker
Worker
```

**Binding Patterns:**

| Pattern | Matches |
|---------|---------|
| `notification.email.*` | `notification.email.order`, `notification.email.welcome` |
| `notification.*.order.*` | `notification.email.order.confirmed`, `notification.sms.order.shipped` |
| `#` | All events |

### Command Queue (Work Queue)

**When to use:**
- Asynchronous task processing
- Load distribution among workers
- Tasks that can be processed in parallel

**Example: Image processing**

```
API
 │
 │  Enqueue job
 ▼
┌───────────────┐
│    Queue      │
│ (image.resize)│
└───────┬───────┘
        │
   ┌────┴────┐
   ▼         ▼
Worker 1   Worker 2
   │         │
   ▼         ▼
Process   Process
Image     Image
```

**Implementation:**

```python
# Producer
async def upload_image(image_data):
    image_id = await storage.save(image_data)

    await queue.publish(
        queue="images.resize",
        message={
            "image_id": image_id,
            "sizes": ["thumbnail", "medium", "large"]
        }
    )

    return {"image_id": image_id, "status": "processing"}

# Consumer (Worker)
@consumer(queue="images.resize", prefetch=5)
async def process_image_resize(message):
    image_id = message["image_id"]
    image_data = await storage.get(image_id)

    for size in message["sizes"]:
        resized = resize_image(image_data, size)
        await storage.save(f"{image_id}_{size}", resized)

    await queue.ack(message)
```

## Distributed Transaction Patterns

### Saga Pattern (Orchestration)

**When to use:**
- Transactions involving multiple services
- Need for compensation on failure
- Eventual consistency is acceptable

**Example: Order creation saga**

```
┌────────────────────────────────────────────────────────────┐
│                    Order Service (Orchestrator)            │
└────────────────────────────────────────────────────────────┘
        │                    │                    │
        │ 1. Reserve         │ 2. Process         │ 3. Confirm
        │    Inventory       │    Payment         │    Reservation
        ▼                    ▼                    ▼
  ┌──────────┐        ┌──────────┐        ┌──────────┐
  │Inventory │        │ Payment  │        │Inventory │
  │ Service  │        │ Service  │        │ Service  │
  └──────────┘        └──────────┘        └──────────┘

Compensation (on failure):
        │                    │
        │ 3'. Refund         │ 2'. Release
        │     Payment        │     Inventory
        ▼                    ▼
  ┌──────────┐        ┌──────────┐
  │ Payment  │        │Inventory │
  │ Service  │        │ Service  │
  └──────────┘        └──────────┘
```

**Implementation:**

```python
class OrderSaga:
    def __init__(self, order_id: str):
        self.order_id = order_id
        self.steps_completed = []

    async def execute(self):
        try:
            # Step 1: Reserve inventory
            await self.reserve_inventory()
            self.steps_completed.append("inventory_reserved")

            # Step 2: Process payment
            await self.process_payment()
            self.steps_completed.append("payment_processed")

            # Step 3: Confirm reservation
            await self.confirm_reservation()
            self.steps_completed.append("reservation_confirmed")

            return {"status": "completed"}

        except Exception as e:
            await self.compensate()
            raise

    async def compensate(self):
        for step in reversed(self.steps_completed):
            if step == "payment_processed":
                await self.refund_payment()
            elif step == "inventory_reserved":
                await self.release_inventory()
```

### Outbox Pattern

**When to use:**
- Ensure event is published along with database operation
- Avoid inconsistency between database and queue
- Atomic transactions (database + event)

**Flow:**

```
┌─────────────────────────────────────────┐
│           Order Service                 │
│  ┌─────────────────────────────────┐   │
│  │         Transaction             │   │
│  │  1. INSERT INTO orders          │   │
│  │  2. INSERT INTO outbox          │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│        Outbox Processor (CDC)           │
│  1. SELECT FROM outbox WHERE sent=false │
│  2. Publish to RabbitMQ                 │
│  3. UPDATE outbox SET sent=true         │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│            RabbitMQ                     │
└─────────────────────────────────────────┘
```

**Implementation:**

```python
async def create_order_with_event(order_data):
    async with database.transaction():
        # 1. Create order
        order = Order(**order_data)
        await order.save()

        # 2. Create event in outbox
        event = OutboxEvent(
            aggregate_type="Order",
            aggregate_id=order.id,
            event_type="OrderCreated",
            payload=order.to_dict()
        )
        await event.save()

    return order
```

## Dead Letter Queue (DLQ)

**When to use:**
- Messages that failed after all attempts
- Processing error analysis
- Manual reprocessing

**Configuration:**

```python
# Queue declaration with DLQ
channel.queue_declare(
    queue="orders.process",
    arguments={
        "x-dead-letter-exchange": "dlx",
        "x-dead-letter-routing-key": "orders.process.dlq",
        "x-message-ttl": 86400000  # 24h
    }
)
```

**Monitoring:**

```bash
# Check messages in DLQ
rabbitmqctl list_queues | grep dlq

# Inspect message
rabbitmqadmin get queue=orders.process.dlq count=1
```

## Idempotency

**When to use:**
- Always in state-modifying operations
- Automatic retry
- At-least-once delivery

**Implementation:**

```python
async def process_payment(idempotency_key: str, order_id: str, amount: int):
    # Check if already processed
    existing = await payment_repository.find_by_idempotency_key(idempotency_key)
    if existing:
        return existing

    # Process payment
    payment = await payment_gateway.charge(order_id, amount)

    # Save with idempotency key
    await payment_repository.save(payment, idempotency_key)

    return payment
```

## Related Links

- [Queue Service](../components/queue-service.md) - Queue infrastructure
- [Order Service](../components/order-service.md) - Saga implementation
- [Data Flow](data-flow.md) - Flow between services
- [Architecture Overview](system-overview.md) - System architecture
