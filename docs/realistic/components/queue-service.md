# Queue Service

## Description

The Queue Service is TechCorp's messaging infrastructure, providing reliable asynchronous communication between microservices through message queues and event topics.

This component encapsulates RabbitMQ complexity, offering abstractions for the most common patterns: work queues, pub/sub with fanout, topic-based routing, and dead letter queues. It also manages the messaging infrastructure, including exchange creation, bindings, and retry policies.

The messaging architecture is fundamental for decoupling TechCorp services, allowing non-critical operations to be processed asynchronously without blocking the user experience.

## Owners

- **Team:** Platform Engineering
- **Tech Lead:** Rafael Lima
- **Slack:** #platform-messaging

## Technology Stack

- Message Broker: RabbitMQ 3.12
- Management: RabbitMQ Management Plugin
- SDK Language: Multiple (Go, Java, Python, Node.js)
- Monitoring: Prometheus Exporter

## Configuration

### Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `RABBITMQ_HOST` | RabbitMQ host | `rabbitmq.internal` |
| `RABBITMQ_PORT` | AMQP port | `5672` |
| `RABBITMQ_MANAGEMENT_PORT` | Management UI port | `15672` |
| `RABBITMQ_USER` | Connection user | - |
| `RABBITMQ_PASSWORD` | Connection password | - |
| `RABBITMQ_VHOST` | Virtual host | `/` |
| `MESSAGE_TTL_MS` | Default message TTL | `86400000` |
| `MAX_RETRIES` | Maximum attempts | `3` |

### Clustering Configuration

```yaml
# rabbitmq.conf
cluster_formation.peer_discovery_backend = k8s
cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
cluster_formation.k8s.address_type = hostname
cluster_partition_handling = pause_minority
```

## Exchange Architecture

### Main Exchanges

| Exchange | Type | Purpose |
|----------|------|---------|
| `events.direct` | direct | Direct events to specific service |
| `events.fanout` | fanout | Broadcast to all consumers |
| `events.topic` | topic | Routing by topic pattern |
| `dlx.events` | direct | Dead letter exchange |

### Routing Patterns

```
# Order events
order.created -> exchange: events.topic, routing_key: order.created
order.confirmed -> exchange: events.topic, routing_key: order.confirmed

# Consumers subscribe with patterns
notification-service: order.*
inventory-service: order.created, order.cancelled
```

## How to Run Locally

```bash
# Start RabbitMQ with Docker
docker run -d --name rabbitmq-local \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin \
  rabbitmq:3.12-management

# Access Management UI
open http://localhost:15672
# Login: admin / admin

# Create basic structure via CLI
docker exec rabbitmq-local rabbitmqadmin declare exchange name=events.topic type=topic
```

### Publish Test Message

```bash
# Via rabbitmqadmin
docker exec rabbitmq-local rabbitmqadmin publish \
  exchange=events.topic \
  routing_key=test.event \
  payload='{"message": "test"}'

# Via SDK (Python example)
import pika
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.basic_publish(exchange='events.topic', routing_key='test.event', body='{"message": "test"}')
```

## Main Queues

| Queue | Exchange | Routing Key | Consumer |
|-------|----------|-------------|----------|
| `notification.email` | events.topic | notification.email.* | notification-service |
| `notification.sms` | events.topic | notification.sms.* | notification-service |
| `inventory.updates` | events.topic | inventory.* | inventory-service |
| `order.processing` | events.direct | - | order-service |
| `payment.webhooks` | events.direct | - | payment-service |

## Retry Policies

### Dead Letter Configuration

```json
{
  "queue": "notification.email",
  "arguments": {
    "x-dead-letter-exchange": "dlx.events",
    "x-dead-letter-routing-key": "notification.email.dlq",
    "x-message-ttl": 86400000
  }
}
```

### Exponential Retry Strategy

```
Attempt 1: Immediate
Attempt 2: After 1 minute
Attempt 3: After 5 minutes
Final failure: Move to DLQ
```

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/queue-service
- **RabbitMQ Management:** https://rabbitmq.techcorp.internal
- **Prometheus Metrics:** https://prometheus.techcorp.internal/targets

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `rabbitmq_queue_messages` | Messages in queue | > 10000 |
| `rabbitmq_queue_messages_unacked` | Unacknowledged messages | > 1000 |
| `rabbitmq_connections` | Active connections | > 500 |
| `rabbitmq_channels` | Active channels | > 1000 |
| `rabbitmq_queue_messages_dlq` | Messages in DLQ | > 0 |

### Configured Alerts

- **QueueBacklogHigh:** Queue with more than 10,000 messages for 5 minutes
- **QueueDLQNotEmpty:** Messages in dead letter queue
- **QueueConsumerDown:** Queue without active consumers
- **RabbitMQClusterPartition:** Network partition in cluster

## Troubleshooting

### Issue: Queue accumulating messages

**Cause:** Slow, stopped, or insufficient consumer.

**Solution:**
1. Check consumers: `rabbitmqctl list_consumers`
2. Check consumer logs
3. Scale consumers: `kubectl scale deployment/<consumer> --replicas=5`
4. If necessary, purge old messages: `rabbitmqadmin purge queue name=<queue>`

### Issue: Messages in DLQ

**Cause:** Processing error after all attempts.

**Solution:**
1. Inspect messages: `rabbitmqadmin get queue=<queue>.dlq count=10`
2. Identify error pattern in `x-death` headers
3. Fix bug in consumer
4. Reprocess messages: move from DLQ to original queue

### Issue: Connections exhausted

**Cause:** Connection leak or too many consumers.

**Solution:**
1. Identify source: `rabbitmqctl list_connections | sort | uniq -c`
2. Check if connections are being closed properly
3. Increase limit: `rabbitmqctl set_vm_memory_high_watermark 0.6`

### Issue: Partitioned cluster

**Cause:** Network problem between cluster nodes.

**Solution:**
1. Check status: `rabbitmqctl cluster_status`
2. Identify problematic node
3. Restart partitioned node: `rabbitmqctl stop_app && rabbitmqctl start_app`
4. If necessary, remove and re-add node to cluster

## Best Practices

### For Producers

- Always use `publisher confirms` to ensure delivery
- Implement retry with exponential backoff
- Use `correlation_id` for tracing

### For Consumers

- Always use `manual ack` (not auto-ack)
- Implement idempotency (messages can be redelivered)
- Configure appropriate `prefetch_count` (not too high)

### Naming Convention

```
Exchanges: {domain}.{type}  -> events.topic, commands.direct
Queues: {service}.{action}  -> notification.email, order.processing
Routing keys: {entity}.{event} -> order.created, user.updated
```

## Related Links

- [Notification Service](notification-service.md) - Main consumer
- [Order Service](order-service.md) - Order event producer
- [Integration Patterns](../architecture/integration-patterns.md) - Messaging patterns
- [System Architecture Overview](../architecture/system-overview.md) - Role in the system
