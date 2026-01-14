# Visão Geral da Arquitetura

## Introdução

A plataforma TechCorp é construída sobre uma arquitetura de microsserviços, projetada para alta disponibilidade, escalabilidade e manutenibilidade. Este documento apresenta uma visão geral do sistema, seus componentes principais e como eles se integram.

## Diagrama de Alto Nível

```
                                    ┌─────────────────┐
                                    │   CDN (Assets)  │
                                    └────────┬────────┘
                                             │
┌──────────────────┐   ┌──────────────────┐  │  ┌──────────────────┐
│   Web Portal     │   │   Mobile App     │  │  │ Admin Dashboard  │
│   (Next.js)      │   │  (React Native)  │  │  │    (React)       │
└────────┬─────────┘   └────────┬─────────┘  │  └────────┬─────────┘
         │                      │            │           │
         └──────────────────────┴────────────┴───────────┘
                                │
                    ┌───────────▼───────────┐
                    │     API Gateway       │
                    │       (Kong)          │
                    └───────────┬───────────┘
                                │
           ┌────────────────────┼────────────────────┐
           │                    │                    │
    ┌──────▼──────┐     ┌───────▼───────┐    ┌──────▼──────┐
    │Auth Service │     │ User Service  │    │Order Service│
    │   (Java)    │     │   (Python)    │    │   (Java)    │
    └──────┬──────┘     └───────┬───────┘    └──────┬──────┘
           │                    │                    │
           │     ┌──────────────┼──────────────┐     │
           │     │              │              │     │
    ┌──────▼─────▼──┐   ┌───────▼───────┐ ┌───▼─────▼──────┐
    │Payment Service│   │  Inventory    │ │  Notification  │
    │     (Go)      │   │   Service     │ │    Service     │
    └───────────────┘   │     (Go)      │ │   (Node.js)    │
                        └───────────────┘ └────────────────┘
                                │
           ┌────────────────────┼────────────────────┐
           │                    │                    │
    ┌──────▼──────┐     ┌───────▼───────┐    ┌──────▼──────┐
    │Search Service│    │ Cache Service │    │Queue Service │
    │ (Python/ES) │     │   (Redis)     │    │ (RabbitMQ)   │
    └─────────────┘     └───────────────┘    └─────────────┘
```

## Componentes do Sistema

### Camada de Apresentação

#### Web Portal
Frontend principal em Next.js para clientes finais. Responsável pela experiência de compra, incluindo navegação de produtos, carrinho e checkout.

**Responsabilidades:**
- Interface de e-commerce
- Server-side rendering para SEO
- Progressive Web App features

**Integrações:**
- API Gateway para todas as chamadas backend
- CDN para assets estáticos

#### Mobile App
Aplicativo móvel em React Native para iOS e Android. Oferece experiência nativa com features como push notifications e biometria.

**Responsabilidades:**
- Experiência mobile nativa
- Push notifications
- Offline mode para catálogo

**Integrações:**
- API Gateway
- Firebase para push notifications

#### Admin Dashboard
Painel administrativo em React para operações internas. Usado por times de suporte, operações e gestão de produtos.

**Responsabilidades:**
- Gestão de pedidos e usuários
- Gestão de catálogo
- Relatórios e dashboards

**Integrações:**
- API Gateway com autenticação admin

### Camada de API

#### API Gateway
Ponto de entrada único para todas as requisições. Implementa roteamento, rate limiting, autenticação de tokens e circuit breaker.

**Responsabilidades:**
- Roteamento de requisições
- Autenticação de tokens JWT
- Rate limiting por cliente
- Load balancing

**Integrações:**
- Auth Service para validação de tokens
- Todos os microsserviços downstream

### Camada de Serviços de Negócio

#### Auth Service
Serviço central de autenticação e autorização. Implementa OAuth 2.0, JWT e MFA.

**Responsabilidades:**
- Autenticação de usuários
- Emissão e validação de tokens
- Gerenciamento de sessões
- Multi-factor authentication

**Dependências:**
- PostgreSQL (credenciais)
- Redis (sessões)
- RabbitMQ (eventos de auditoria)

#### User Service
Gerenciamento do ciclo de vida de usuários, incluindo dados cadastrais, preferências e endereços.

**Responsabilidades:**
- CRUD de usuários
- Gerenciamento de preferências
- Histórico de alterações

**Dependências:**
- PostgreSQL (dados)
- Redis (cache)
- Auth Service (validação de e-mail)
- Notification Service (confirmações)

#### Order Service
Orquestra o ciclo de vida completo de pedidos, desde criação até entrega.

**Responsabilidades:**
- Criação e gerenciamento de pedidos
- Máquina de estados de pedido
- Coordenação de saga para transações distribuídas

**Dependências:**
- PostgreSQL (pedidos)
- Redis (cache)
- Inventory Service (reservas)
- Payment Service (pagamentos)
- Notification Service (status updates)
- RabbitMQ (eventos)

#### Payment Service
Processamento de transações financeiras com múltiplos provedores de pagamento.

**Responsabilidades:**
- Processamento de pagamentos
- Integração com gateways
- Gerenciamento de estornos

**Dependências:**
- PostgreSQL (transações)
- RabbitMQ (eventos)
- Provedores externos (Stripe, PagSeguro, etc.)

#### Inventory Service
Controle de estoque em tempo real com suporte a múltiplos depósitos.

**Responsabilidades:**
- Controle de disponibilidade
- Reservas temporárias
- Sincronização com sistemas legados

**Dependências:**
- PostgreSQL (estoque)
- Redis (contadores em tempo real)
- RabbitMQ (eventos)

#### Notification Service
Envio de comunicações multi-canal: e-mail, SMS, push e in-app.

**Responsabilidades:**
- Envio de notificações
- Gerenciamento de templates
- Controle de preferências de opt-out

**Dependências:**
- PostgreSQL (templates, histórico)
- Redis (rate limiting)
- RabbitMQ (fila de envios)
- Provedores externos (SendGrid, Twilio, etc.)

### Camada de Infraestrutura

#### Search Service
Motor de busca full-text para produtos e pedidos, baseado em Elasticsearch.

**Responsabilidades:**
- Indexação de produtos
- Busca full-text com relevância
- Autocomplete e sugestões

**Dependências:**
- Elasticsearch (índices)
- Redis (cache de queries)
- RabbitMQ (eventos de atualização)

#### Cache Service
Camada de caching distribuído baseada em Redis.

**Responsabilidades:**
- Cache de queries frequentes
- Armazenamento de sessões
- Rate limiting
- Locks distribuídos

#### Queue Service
Infraestrutura de mensageria baseada em RabbitMQ para comunicação assíncrona.

**Responsabilidades:**
- Filas de mensagens
- Pub/sub de eventos
- Dead letter queues
- Retry automático

### Camada de Dados

#### PostgreSQL
Banco de dados relacional principal, hospedado no Amazon RDS com Multi-AZ.

**Schemas:**
- `auth` - Credenciais e tokens
- `users` - Dados de usuários
- `orders` - Pedidos e itens
- `payments` - Transações
- `inventory` - Estoque
- `products` - Catálogo

#### Redis
Armazenamento em memória para cache, sessões e rate limiting.

**Clusters:**
- `sessions` - Sessões de usuário
- `queries` - Cache de API
- `ratelimit` - Rate limiting
- `locks` - Locks distribuídos

#### Elasticsearch
Motor de busca para catálogo e pedidos.

**Índices:**
- `products` - Catálogo de produtos
- `orders` - Histórico de pedidos (busca)

### Camada de Orquestração

#### Kubernetes Cluster
Cluster EKS para orquestração de containers.

**Node Groups:**
- `general` - Workloads gerais
- `compute` - CPU-intensive
- `memory` - Memory-intensive
- `spot` - Batch jobs

#### Monitoring Stack
Prometheus, Grafana e Alertmanager para observabilidade.

## Fluxos Principais

### Fluxo de Autenticação

1. Cliente envia credenciais para API Gateway
2. API Gateway roteia para Auth Service
3. Auth Service valida credenciais no PostgreSQL
4. Auth Service emite JWT
5. JWT é retornado ao cliente
6. Requisições subsequentes incluem JWT
7. API Gateway valida JWT com Auth Service

### Fluxo de Pedido

1. Cliente finaliza checkout no Web Portal
2. API Gateway recebe requisição
3. Order Service cria pedido (status: PENDING)
4. Order Service solicita reserva ao Inventory Service
5. Inventory Service reserva estoque (TTL: 15 min)
6. Order Service solicita pagamento ao Payment Service
7. Payment Service processa com provedor externo
8. Payment Service publica evento de sucesso/falha
9. Order Service atualiza status
10. Inventory Service confirma ou libera reserva
11. Notification Service envia confirmação ao cliente

### Fluxo de Busca

1. Cliente digita termo de busca
2. API Gateway recebe requisição
3. Search Service verifica cache no Redis
4. Se cache miss, consulta Elasticsearch
5. Resultados são cacheados e retornados
6. Eventos de indexação atualizam índices assincronamente

## Características de Resiliência

### Circuit Breaker
Todos os serviços implementam circuit breaker para chamadas downstream, evitando cascata de falhas.

### Retry com Backoff
Operações que podem falhar temporariamente usam retry com backoff exponencial.

### Graceful Degradation
Features não-críticas podem ser desabilitadas sem impactar o fluxo principal.

### Idempotência
Operações críticas (pagamentos, reservas) são idempotentes para suportar retry seguro.

## Escalabilidade

### Horizontal
- Todos os serviços stateless podem escalar horizontalmente
- HPA configurado para auto-scaling baseado em CPU/memória
- Workers de fila escalam independentemente

### Vertical
- PostgreSQL pode escalar verticalmente (instâncias maiores)
- Redis pode escalar horizontalmente (cluster mode)

## Segurança

- Todas as comunicações são criptografadas (TLS)
- Autenticação via JWT com refresh tokens
- Secrets gerenciados pelo AWS Secrets Manager
- Network policies isolam serviços no Kubernetes
- PCI-DSS compliance para dados de pagamento

## Links Relacionados

### Componentes
- [API Gateway](../components/api-gateway.md)
- [Auth Service](../components/auth-service.md)
- [User Service](../components/user-service.md)
- [Order Service](../components/order-service.md)
- [Payment Service](../components/payment-service.md)
- [Inventory Service](../components/inventory-service.md)
- [Notification Service](../components/notification-service.md)
- [Search Service](../components/search-service.md)
- [Cache Service](../components/cache-service.md)
- [Queue Service](../components/queue-service.md)
- [Kubernetes Cluster](../components/kubernetes-cluster.md)
- [Database PostgreSQL](../components/database-postgres.md)
- [Monitoring Stack](../components/monitoring-stack.md)

### Arquitetura
- [Fluxo de Dados](data-flow.md)
- [Modelo de Segurança](security-model.md)
- [Padrões de Integração](integration-patterns.md)
