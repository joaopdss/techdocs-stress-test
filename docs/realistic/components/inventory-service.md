# Inventory Service

## Descrição

O Inventory Service é o microsserviço responsável pelo controle de estoque da TechCorp. Este componente gerencia a disponibilidade de produtos em tempo real, realizando reservas, baixas e reposições de estoque em múltiplos depósitos e centros de distribuição.

O serviço implementa um modelo de estoque com reserva temporal, onde itens são reservados durante o processo de checkout e liberados automaticamente se o pagamento não for confirmado dentro de um período configurável. Isso evita overselling enquanto mantém a disponibilidade para outros clientes.

Para operações de alto volume, o inventory-service utiliza contadores distribuídos com Redis para garantir performance e consistência eventual. Sincronizações periódicas reconciliam os contadores com o banco de dados principal.

## Responsáveis

- **Time:** Supply Chain Tech
- **Tech Lead:** Marcos Ribeiro
- **Slack:** #supply-inventory

## Stack Tecnológica

- Linguagem: Go 1.21
- Framework: Chi Router
- Banco de dados: PostgreSQL 15
- Cache: Redis 7 (contadores distribuídos)
- Message Broker: RabbitMQ 3.12

## Configuração

### Variáveis de Ambiente

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| `INVENTORY_PORT` | Porta HTTP do serviço | `3006` |
| `DATABASE_URL` | Connection string PostgreSQL | - |
| `REDIS_URL` | URL do Redis | - |
| `RABBITMQ_URL` | URL do RabbitMQ | - |
| `RESERVATION_TTL_MINUTES` | Tempo de vida da reserva | `15` |
| `SYNC_INTERVAL_SECONDS` | Intervalo de sincronização Redis->DB | `60` |
| `LOW_STOCK_THRESHOLD` | Limite para alerta de estoque baixo | `10` |

### Configuração de Depósitos

```yaml
# config/warehouses.yaml
warehouses:
  - id: wh-sp-001
    name: CD São Paulo
    region: southeast
    priority: 1
  - id: wh-rj-001
    name: CD Rio de Janeiro
    region: southeast
    priority: 2
  - id: wh-pr-001
    name: CD Curitiba
    region: south
    priority: 1
```

## Como Executar Localmente

```bash
# Clonar o repositório
git clone git@github.com:techcorp/inventory-service.git
cd inventory-service

# Subir dependências
docker-compose up -d postgres redis rabbitmq

# Compilar
go build -o inventory-service ./cmd/server

# Executar migrações
./inventory-service migrate

# Seed de dados de teste
./inventory-service seed --warehouses --products

# Iniciar o serviço
./inventory-service serve

# Verificar saúde
curl http://localhost:3006/health
```

### Consultar Estoque

```bash
# Verificar disponibilidade de um produto
curl http://localhost:3006/api/inventory/products/uuid/availability

# Reservar estoque
curl -X POST http://localhost:3006/api/inventory/reservations \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": "uuid",
    "items": [
      {"product_id": "uuid", "quantity": 2, "warehouse_id": "wh-sp-001"}
    ]
  }'
```

## Modelo de Dados

### Entidade StockLevel

```go
type StockLevel struct {
    ProductID    uuid.UUID
    WarehouseID  string
    Available    int       // Quantidade disponível para venda
    Reserved     int       // Quantidade reservada (checkout em andamento)
    Incoming     int       // Quantidade em trânsito de reposição
    UpdatedAt    time.Time
}
```

### Entidade Reservation

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

## Operações de Estoque

### Fluxo de Reserva

```
1. Checkout inicia -> Cria reserva com TTL de 15 min
2. Pagamento confirmado -> Converte reserva em baixa de estoque
3. Pagamento falha/timeout -> Reserva expira, estoque liberado
```

### Estratégia de Alocação

O serviço seleciona o depósito automaticamente baseado em:

1. **Disponibilidade:** Depósito deve ter estoque suficiente
2. **Proximidade:** Prioriza depósito mais próximo do CEP de entrega
3. **Prioridade:** Configuração de prioridade por região
4. **Balanceamento:** Distribui carga entre depósitos

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/inventory-service
- **Métricas Prometheus:** https://prometheus.techcorp.internal/targets
- **Logs:** Enviados para Elasticsearch via Fluentd

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `inventory_reservations_total` | Total de reservas | - |
| `inventory_reservations_expired` | Reservas expiradas | > 10% |
| `inventory_stock_level` | Nível de estoque por produto | < threshold |
| `inventory_sync_duration_seconds` | Duração da sincronização | > 10s |
| `inventory_oversell_events` | Eventos de oversell | > 0 |

### Alertas Configurados

- **InventoryLowStock:** Produto com estoque abaixo do threshold
- **InventoryOversell:** Venda acima do estoque disponível detectada
- **InventorySyncFailed:** Sincronização Redis->DB falhou
- **InventoryReservationExpiredHigh:** Taxa de expiração acima de 10%

## Troubleshooting

### Problema: Estoque mostrando disponível mas pedido falhando

**Causa:** Inconsistência entre contador Redis e banco de dados.

**Solução:**
1. Forçar sincronização: `POST /admin/inventory/sync`
2. Verificar valores: `GET /admin/inventory/products/<id>/debug`
3. Se necessário, resetar contador: `POST /admin/inventory/products/<id>/reset-counter`

### Problema: Reservas não expirando

**Causa:** Worker de expiração parado ou problema no Redis.

**Solução:**
1. Verificar worker: `kubectl logs -l app=inventory-expiration-worker`
2. Verificar TTL no Redis: `redis-cli TTL reservation:<id>`
3. Expirar manualmente se urgente: `POST /admin/reservations/<id>/expire`

### Problema: Estoque negativo detectado

**Causa:** Race condition em operações concorrentes ou bug no contador.

**Solução:**
1. Identificar produto afetado nos logs
2. Bloquear vendas temporariamente: `POST /admin/products/<id>/block-sales`
3. Reconciliar estoque manualmente com inventário físico
4. Desbloquear após correção

### Problema: Sincronização lenta

**Causa:** Volume alto de produtos ou conexão lenta com banco.

**Solução:**
1. Verificar métricas de latência do PostgreSQL
2. Aumentar paralelismo da sincronização: `SYNC_WORKERS=10`
3. Considerar sincronização parcial por warehouse

## Eventos Consumidos

| Evento | Origem | Ação |
|--------|--------|------|
| `order.created` | order-service | Criar reserva de estoque |
| `payment.confirmed` | payment-service | Confirmar reserva (baixa) |
| `payment.failed` | payment-service | Cancelar reserva |
| `order.cancelled` | order-service | Estornar estoque |

## Eventos Publicados

| Evento | Exchange | Descrição |
|--------|----------|-----------|
| `inventory.reserved` | `inventory.events` | Estoque reservado |
| `inventory.confirmed` | `inventory.events` | Reserva confirmada |
| `inventory.released` | `inventory.events` | Estoque liberado |
| `inventory.low_stock` | `inventory.events` | Alerta de estoque baixo |
| `inventory.out_of_stock` | `inventory.events` | Produto sem estoque |

## Links Relacionados

- [Order Service](order-service.md) - Processamento de pedidos
- [API de Produtos](../apis/products-api.md) - Caminhos da API de produtos
- [Fluxo de Dados](../architecture/data-flow.md) - Integração de estoque no fluxo
- [Problemas de Performance](../troubleshooting/performance-issues.md) - Otimizações
