# Fluxo de Dados

## Visão Geral

Este documento descreve como os dados fluem através dos sistemas da TechCorp, desde a interação do usuário até a persistência final. Compreender esses fluxos é essencial para diagnóstico de problemas e otimização de performance.

## Fluxo de Pedido Completo

### Diagrama de Sequência

```
Cliente    Gateway    Order    Inventory   Payment    Notification
   │          │         │          │          │            │
   ├─────────►│         │          │          │            │
   │  POST    │         │          │          │            │
   │ /orders  ├────────►│          │          │            │
   │          │ Criar   ├─────────►│          │            │
   │          │ pedido  │ Reservar │          │            │
   │          │         │ estoque  │          │            │
   │          │         │◄─────────┤          │            │
   │          │         │   OK     │          │            │
   │          │         ├──────────┼─────────►│            │
   │          │         │          │ Processar│            │
   │          │         │          │ pagamento│            │
   │          │         │◄─────────┼──────────┤            │
   │          │         │          │   OK     │            │
   │          │         ├─────────►│          │            │
   │          │         │ Confirmar│          │            │
   │          │         │ reserva  │          │            │
   │          │         ├──────────┼──────────┼───────────►│
   │          │         │          │          │   Enviar   │
   │          │         │          │          │   e-mail   │
   │◄─────────┤◄────────┤          │          │            │
   │   200    │  Pedido │          │          │            │
   │          │  criado │          │          │            │
```

### Detalhamento das Etapas

#### 1. Criação do Pedido

**Origem:** Web Portal / Mobile App
**Destino:** Order Service via API Gateway

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

**Transformações:**
1. API Gateway valida JWT
2. API Gateway adiciona headers de contexto
3. Order Service valida dados de entrada
4. Order Service cria registro em `orders.orders`

#### 2. Reserva de Estoque

**Origem:** Order Service
**Destino:** Inventory Service

```json
{
  "order_id": "uuid",
  "items": [
    {"product_id": "uuid", "quantity": 2}
  ],
  "reservation_ttl_minutes": 15
}
```

**Transformações:**
1. Inventory Service verifica disponibilidade
2. Inventory Service cria reserva com TTL
3. Inventory Service atualiza contadores Redis
4. Inventory Service registra em `inventory.reservations`

**Dados Persistidos:**
- Tabela: `inventory.reservations`
- Redis: `inventory:product:{id}:available` (decremento)

#### 3. Processamento do Pagamento

**Origem:** Order Service
**Destino:** Payment Service

```json
{
  "order_id": "uuid",
  "amount": 29990,
  "currency": "BRL",
  "method": "credit_card",
  "card_token": "tok_xxx"
}
```

**Transformações:**
1. Payment Service cria transação (status: PENDING)
2. Payment Service envia para provedor externo
3. Provedor retorna autorização
4. Payment Service atualiza transação (status: APPROVED)
5. Payment Service publica evento `payment.confirmed`

**Dados Persistidos:**
- Tabela: `payments.transactions`
- Evento: `payment.confirmed` no RabbitMQ

#### 4. Confirmação da Reserva

**Origem:** Order Service (via evento)
**Destino:** Inventory Service

```json
{
  "event": "payment.confirmed",
  "order_id": "uuid"
}
```

**Transformações:**
1. Inventory Service recebe evento
2. Inventory Service converte reserva em baixa
3. Inventory Service atualiza estoque real

**Dados Persistidos:**
- Tabela: `inventory.stock_movements`
- `inventory.reservations` atualizada (status: CONFIRMED)

#### 5. Notificação ao Cliente

**Origem:** Order Service (via evento)
**Destino:** Notification Service

```json
{
  "event": "order.confirmed",
  "user_id": "uuid",
  "order_id": "uuid",
  "order_number": "TC-2024-001234"
}
```

**Transformações:**
1. Notification Service recebe evento
2. Notification Service carrega template
3. Notification Service renderiza com dados do pedido
4. Notification Service envia via provedor (SendGrid)

**Dados Persistidos:**
- Tabela: `notifications.delivery_log`

## Fluxo de Autenticação

```
Cliente    Gateway    Auth      Redis     PostgreSQL
   │          │         │         │          │
   ├─────────►│         │         │          │
   │  POST    │         │         │          │
   │ /login   ├────────►│         │          │
   │          │         ├─────────┼─────────►│
   │          │         │         │  Buscar  │
   │          │         │         │  usuário │
   │          │         │◄────────┼──────────┤
   │          │         │         │          │
   │          │         │ Validar │          │
   │          │         │ senha   │          │
   │          │         │         │          │
   │          │         ├────────►│          │
   │          │         │ Criar   │          │
   │          │         │ sessão  │          │
   │          │         │◄────────┤          │
   │          │         │         │          │
   │          │         │ Gerar   │          │
   │          │         │ JWT     │          │
   │◄─────────┤◄────────┤         │          │
   │   JWT    │         │         │          │
```

### Dados do Token JWT

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

### Armazenamento de Sessão

```
Redis Key: session:{session_id}
TTL: 7 dias

Value: {
  "user_id": "uuid",
  "refresh_token": "xxx",
  "device_info": {...},
  "created_at": "2024-01-15T10:00:00Z"
}
```

## Fluxo de Busca de Produtos

```
Cliente    Gateway    Search    Redis    Elasticsearch
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

### Query Elasticsearch

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "camiseta azul",
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

## Fluxo de Indexação

```
Catalog    RabbitMQ    Search    Elasticsearch
Service
   │          │          │            │
   │ Produto  │          │            │
   │ atualizado          │            │
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

### Evento de Atualização

```json
{
  "event": "product.updated",
  "product_id": "uuid",
  "changes": ["price", "stock"],
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### Documento Indexado

```json
{
  "id": "uuid",
  "title": "Camiseta Básica Algodão",
  "description": "Camiseta 100% algodão...",
  "price": 89.90,
  "original_price": 119.90,
  "category": "camisetas",
  "brand": "techcorp-basics",
  "attributes": {
    "cor": ["azul", "vermelho", "preto"],
    "tamanho": ["p", "m", "g", "gg"]
  },
  "in_stock": true,
  "popularity_score": 0.85,
  "updated_at": "2024-01-15T10:30:00Z"
}
```

## Consistência de Dados

### Padrão Saga

Para operações que envolvem múltiplos serviços, usamos o padrão Saga com compensação:

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

Eventos críticos são armazenados para auditoria e replay:

```
orders.order_events:
- order_id: uuid
- event_type: CREATED | PAID | SHIPPED | DELIVERED
- event_data: {...}
- created_at: timestamp
```

## Replicação de Dados

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

## Links Relacionados

- [Visão Geral da Arquitetura](system-overview.md) - Arquitetura do sistema
- [Padrões de Integração](integration-patterns.md) - Padrões de mensageria
- [Order Service](../components/order-service.md) - Serviço de pedidos
- [Inventory Service](../components/inventory-service.md) - Serviço de estoque
