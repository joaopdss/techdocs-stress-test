# Padrões de Integração

## Visão Geral

Este documento descreve os padrões de integração utilizados na comunicação entre microsserviços da TechCorp. A escolha do padrão adequado depende dos requisitos de consistência, latência e resiliência de cada caso de uso.

## Padrões Síncronos

### Request-Response via REST

**Quando usar:**
- Operações que requerem resposta imediata
- Queries simples de dados
- Validações em tempo real

**Exemplo: Validar disponibilidade de estoque**

```
Order Service                 Inventory Service
     │                              │
     │  GET /inventory/{id}/check   │
     ├─────────────────────────────►│
     │                              │
     │     { available: true }      │
     │◄─────────────────────────────┤
```

**Implementação:**

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

**Quando usar:**
- Proteger contra falhas em cascata
- Serviços com histórico de instabilidade
- Operações críticas com fallback

**Estados:**

```
CLOSED ──────────► OPEN ──────────► HALF-OPEN
   │                 │                  │
   │ N falhas        │ Timeout          │ 1 sucesso
   │ consecutivas    │ expirado         │
   │                 │                  │
   └─────────────────┴──────────────────┘
```

**Implementação:**

```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30)
async def process_payment(order_id: str, amount: int):
    return await payment_client.charge(order_id, amount)

# Fallback quando circuito aberto
async def process_payment_with_fallback(order_id: str, amount: int):
    try:
        return await process_payment(order_id, amount)
    except CircuitBreakerError:
        # Enfileirar para processamento posterior
        await queue.publish("payments.retry", {"order_id": order_id, "amount": amount})
        return {"status": "queued"}
```

### Retry com Backoff Exponencial

**Quando usar:**
- Falhas transitórias de rede
- Timeouts ocasionais
- Rate limiting temporário

**Estratégia:**

```
Tentativa 1: imediata
Tentativa 2: após 1s
Tentativa 3: após 2s
Tentativa 4: após 4s
Tentativa 5: após 8s (máximo)
```

**Implementação:**

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=1, max=8)
)
async def call_external_service(payload):
    return await http_client.post(url, json=payload)
```

## Padrões Assíncronos

### Publish-Subscribe (Fanout)

**Quando usar:**
- Múltiplos consumidores interessados no mesmo evento
- Desacoplamento entre produtor e consumidores
- Eventos de domínio (domain events)

**Exemplo: Evento de pedido criado**

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

**Implementação:**

```python
# Produtor (Order Service)
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

# Consumidor (Notification Service)
@consumer(queue="notification.order-events")
async def handle_order_created(message):
    if message["event"] == "order.created":
        await send_order_confirmation_email(
            user_id=message["user_id"],
            order_id=message["order_id"]
        )
```

### Topic-Based Routing

**Quando usar:**
- Consumidores interessados em subconjunto de eventos
- Roteamento baseado em padrões
- Filtro de eventos no broker

**Exemplo: Roteamento por tipo de notificação**

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
| `#` | Todos os eventos |

### Command Queue (Work Queue)

**Quando usar:**
- Processamento assíncrono de tarefas
- Distribuição de carga entre workers
- Tarefas que podem ser processadas em paralelo

**Exemplo: Processamento de imagens**

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

**Implementação:**

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

## Padrões de Transação Distribuída

### Saga Pattern (Orquestração)

**Quando usar:**
- Transações que envolvem múltiplos serviços
- Necessidade de compensação em caso de falha
- Consistência eventual é aceitável

**Exemplo: Saga de criação de pedido**

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

Compensação (em caso de falha):
        │                    │
        │ 3'. Refund         │ 2'. Release
        │     Payment        │     Inventory
        ▼                    ▼
  ┌──────────┐        ┌──────────┐
  │ Payment  │        │Inventory │
  │ Service  │        │ Service  │
  └──────────┘        └──────────┘
```

**Implementação:**

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

**Quando usar:**
- Garantir que evento é publicado junto com operação de banco
- Evitar inconsistência entre banco e fila
- Transações atômicas (banco + evento)

**Fluxo:**

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

**Implementação:**

```python
async def create_order_with_event(order_data):
    async with database.transaction():
        # 1. Criar pedido
        order = Order(**order_data)
        await order.save()

        # 2. Criar evento no outbox
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

**Quando usar:**
- Mensagens que falharam após todas as tentativas
- Análise de erros de processamento
- Reprocessamento manual

**Configuração:**

```python
# Declaração da fila com DLQ
channel.queue_declare(
    queue="orders.process",
    arguments={
        "x-dead-letter-exchange": "dlx",
        "x-dead-letter-routing-key": "orders.process.dlq",
        "x-message-ttl": 86400000  # 24h
    }
)
```

**Monitoramento:**

```bash
# Verificar mensagens na DLQ
rabbitmqctl list_queues | grep dlq

# Inspecionar mensagem
rabbitmqadmin get queue=orders.process.dlq count=1
```

## Idempotência

**Quando usar:**
- Sempre em operações que modificam estado
- Retry automático
- At-least-once delivery

**Implementação:**

```python
async def process_payment(idempotency_key: str, order_id: str, amount: int):
    # Verificar se já foi processado
    existing = await payment_repository.find_by_idempotency_key(idempotency_key)
    if existing:
        return existing

    # Processar pagamento
    payment = await payment_gateway.charge(order_id, amount)

    # Salvar com idempotency key
    await payment_repository.save(payment, idempotency_key)

    return payment
```

## Links Relacionados

- [Queue Service](../components/queue-service.md) - Infraestrutura de filas
- [Order Service](../components/order-service.md) - Implementação de Saga
- [Fluxo de Dados](data-flow.md) - Fluxo entre serviços
- [Visão Geral da Arquitetura](system-overview.md) - Arquitetura do sistema
