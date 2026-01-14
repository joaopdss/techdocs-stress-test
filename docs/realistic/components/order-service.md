# Order Service

## Descrição

O Order Service é o microsserviço central de processamento de pedidos da TechCorp. Este componente orquestra todo o ciclo de vida de um pedido, desde a criação até a conclusão, coordenando interações com inventory-service, payment-service e notification-service.

O serviço implementa uma máquina de estados robusta para gerenciar as transições de status do pedido (PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED). Cada transição dispara eventos que são consumidos por outros serviços para manter a consistência do sistema.

Para garantir a resiliência, o order-service utiliza o padrão Saga para transações distribuídas, permitindo compensação automática em caso de falhas em qualquer etapa do processo de pedido.

## Responsáveis

- **Time:** Commerce Engineering
- **Tech Lead:** Juliana Almeida
- **Slack:** #commerce-orders

## Stack Tecnológica

- Linguagem: Java 21
- Framework: Spring Boot 3.2 + Spring State Machine
- Banco de dados: PostgreSQL 15
- Cache: Redis 7
- Message Broker: RabbitMQ 3.12
- Search: Elasticsearch 8.11

## Configuração

### Variáveis de Ambiente

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| `ORDER_SERVICE_PORT` | Porta HTTP do serviço | `3003` |
| `DATABASE_URL` | Connection string PostgreSQL | - |
| `REDIS_URL` | URL do Redis | - |
| `RABBITMQ_URL` | URL do RabbitMQ | - |
| `ELASTICSEARCH_URL` | URL do Elasticsearch | - |
| `INVENTORY_SERVICE_URL` | URL do inventory-service | - |
| `PAYMENT_SERVICE_URL` | URL do payment-service | - |
| `NOTIFICATION_SERVICE_URL` | URL do notification-service | - |
| `SAGA_TIMEOUT_SECONDS` | Timeout para compensação de saga | `300` |

### Configuração de State Machine

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

## Como Executar Localmente

```bash
# Clonar o repositório
git clone git@github.com:techcorp/order-service.git
cd order-service

# Subir dependências
docker-compose up -d postgres redis rabbitmq elasticsearch

# Compilar
./mvnw clean package -DskipTests

# Executar migrações
./mvnw flyway:migrate

# Iniciar o serviço
./mvnw spring-boot:run -Dspring.profiles.active=local

# Verificar saúde
curl http://localhost:3003/actuator/health
```

### Criar Pedido de Teste

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

## Máquina de Estados do Pedido

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

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/order-service
- **Métricas Prometheus:** https://prometheus.techcorp.internal/targets
- **Logs:** Enviados para Elasticsearch via Fluentd

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `orders_created_total` | Total de pedidos criados | - |
| `orders_completed_total` | Pedidos finalizados | - |
| `orders_cancelled_total` | Pedidos cancelados | > 5% |
| `order_processing_duration_seconds` | Tempo de processamento | > 30s |
| `saga_compensation_total` | Compensações de saga | > 1/min |

### Alertas Configurados

- **OrderServiceHighCancellationRate:** Taxa de cancelamento acima de 5%
- **OrderSagaCompensationHigh:** Mais de 10 compensações por minuto
- **OrderProcessingTimeout:** Pedidos parados em PROCESSING por mais de 1 hora

## Troubleshooting

### Problema: Pedido preso no status PAYMENT_PENDING

**Causa:** Timeout na comunicação com payment-service ou webhook não recebido.

**Solução:**
1. Verificar status no payment-service: `GET /payments?order_id=<id>`
2. Se pagamento confirmado, disparar evento manualmente: `POST /admin/orders/<id>/events/payment_received`
3. Se pagamento não existe, cancelar pedido: `POST /admin/orders/<id>/cancel`

### Problema: Saga executando compensação inesperada

**Causa:** Timeout de saga configurado muito baixo ou serviço downstream lento.

**Solução:**
1. Verificar logs da saga: filtrar por `saga.order.<order_id>`
2. Identificar qual step falhou no histórico da saga
3. Aumentar timeout se necessário ou investigar serviço lento

### Problema: Busca de pedidos lenta

**Causa:** Índice do Elasticsearch desatualizado ou query mal otimizada.

**Solução:**
1. Forçar reindexação: `POST /admin/orders/reindex`
2. Verificar mapeamento do índice
3. Analisar query no Kibana Dev Tools

## Eventos Consumidos

| Evento | Origem | Ação |
|--------|--------|------|
| `payment.confirmed` | payment-service | Transiciona para PAYMENT_CONFIRMED |
| `payment.failed` | payment-service | Transiciona para CANCELLED |
| `inventory.reserved` | inventory-service | Marca itens como reservados |
| `shipping.delivered` | logistics | Transiciona para DELIVERED |

## Eventos Publicados

| Evento | Exchange | Descrição |
|--------|----------|-----------|
| `order.created` | `order.events` | Novo pedido criado |
| `order.confirmed` | `order.events` | Pedido confirmado |
| `order.shipped` | `order.events` | Pedido enviado |
| `order.delivered` | `order.events` | Pedido entregue |
| `order.cancelled` | `order.events` | Pedido cancelado |

## Links Relacionados

- [API de Pedidos](../apis/orders-api.md) - Recursos REST disponíveis
- [Payment Service](payment-service.md) - Processamento de pagamentos
- [Inventory Service](inventory-service.md) - Controle de estoque
- [Notification Service](notification-service.md) - Notificações de status
- [Fluxo de Dados](../architecture/data-flow.md) - Fluxo completo do pedido
