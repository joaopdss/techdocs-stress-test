# Architecture Overview

## Introduction

The TechCorp platform is built on a microservices architecture, designed for high availability, scalability, and maintainability. This document presents a system overview, its main components, and how they integrate.

## High-Level Diagram

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

## System Components

### Presentation Layer

#### Web Portal
Main frontend in Next.js for end customers. Responsible for the shopping experience, including product navigation, cart, and checkout.

**Responsibilities:**
- E-commerce interface
- Server-side rendering for SEO
- Progressive Web App features

**Integrations:**
- API Gateway for all backend calls
- CDN for static assets

#### Mobile App
Mobile application in React Native for iOS and Android. Offers native experience with features like push notifications and biometrics.

**Responsibilities:**
- Native mobile experience
- Push notifications
- Offline mode for catalog

**Integrations:**
- API Gateway
- Firebase for push notifications

#### Admin Dashboard
Administrative panel in React for internal operations. Used by support, operations, and product management teams.

**Responsibilities:**
- Order and user management
- Catalog management
- Reports and dashboards

**Integrations:**
- API Gateway with admin authentication

### API Layer

#### API Gateway
Single entry point for all requests. Implements routing, rate limiting, token authentication, and circuit breaker.

**Responsibilities:**
- Request routing
- JWT token authentication
- Rate limiting per client
- Load balancing

**Integrations:**
- Auth Service for token validation
- All downstream microservices

### Business Services Layer

#### Auth Service
Central authentication and authorization service. Implements OAuth 2.0, JWT, and MFA.

**Responsibilities:**
- User authentication
- Token issuance and validation
- Session management
- Multi-factor authentication

**Dependencies:**
- PostgreSQL (credentials)
- Redis (sessions)
- RabbitMQ (audit events)

#### User Service
User lifecycle management, including registration data, preferences, and addresses.

**Responsibilities:**
- User CRUD
- Preferences management
- Change history

**Dependencies:**
- PostgreSQL (data)
- Redis (cache)
- Auth Service (email validation)
- Notification Service (confirmations)

#### Order Service
Orchestrates the complete order lifecycle, from creation to delivery.

**Responsibilities:**
- Order creation and management
- Order state machine
- Saga coordination for distributed transactions

**Dependencies:**
- PostgreSQL (orders)
- Redis (cache)
- Inventory Service (reservations)
- Payment Service (payments)
- Notification Service (status updates)
- RabbitMQ (events)

#### Payment Service
Financial transaction processing with multiple payment providers.

**Responsibilities:**
- Payment processing
- Gateway integration
- Refund management

**Dependencies:**
- PostgreSQL (transactions)
- RabbitMQ (events)
- External providers (Stripe, PagSeguro, etc.)

#### Inventory Service
Real-time inventory control with support for multiple warehouses.

**Responsibilities:**
- Availability control
- Temporary reservations
- Synchronization with legacy systems

**Dependencies:**
- PostgreSQL (inventory)
- Redis (real-time counters)
- RabbitMQ (events)

#### Notification Service
Multi-channel communication delivery: email, SMS, push, and in-app.

**Responsibilities:**
- Notification delivery
- Template management
- Opt-out preferences control

**Dependencies:**
- PostgreSQL (templates, history)
- Redis (rate limiting)
- RabbitMQ (send queue)
- External providers (SendGrid, Twilio, etc.)

### Infrastructure Layer

#### Search Service
Full-text search engine for products and orders, based on Elasticsearch.

**Responsibilities:**
- Product indexing
- Full-text search with relevance
- Autocomplete and suggestions

**Dependencies:**
- Elasticsearch (indexes)
- Redis (query cache)
- RabbitMQ (update events)

#### Cache Service
Distributed caching layer based on Redis.

**Responsibilities:**
- Frequent query caching
- Session storage
- Rate limiting
- Distributed locks

#### Queue Service
Messaging infrastructure based on RabbitMQ for asynchronous communication.

**Responsibilities:**
- Message queues
- Event pub/sub
- Dead letter queues
- Automatic retry

### Data Layer

#### PostgreSQL
Main relational database, hosted on Amazon RDS with Multi-AZ.

**Schemas:**
- `auth` - Credentials and tokens
- `users` - User data
- `orders` - Orders and items
- `payments` - Transactions
- `inventory` - Stock
- `products` - Catalog

#### Redis
In-memory storage for caching, sessions, and rate limiting.

**Clusters:**
- `sessions` - User sessions
- `queries` - API cache
- `ratelimit` - Rate limiting
- `locks` - Distributed locks

#### Elasticsearch
Search engine for catalog and orders.

**Indexes:**
- `products` - Product catalog
- `orders` - Order history (search)

### Orchestration Layer

#### Kubernetes Cluster
EKS cluster for container orchestration.

**Node Groups:**
- `general` - General workloads
- `compute` - CPU-intensive
- `memory` - Memory-intensive
- `spot` - Batch jobs

#### Monitoring Stack
Prometheus, Grafana, and Alertmanager for observability.

## Main Flows

### Authentication Flow

1. Client sends credentials to API Gateway
2. API Gateway routes to Auth Service
3. Auth Service validates credentials in PostgreSQL
4. Auth Service issues JWT
5. JWT is returned to client
6. Subsequent requests include JWT
7. API Gateway validates JWT with Auth Service

### Order Flow

1. Client completes checkout on Web Portal
2. API Gateway receives request
3. Order Service creates order (status: PENDING)
4. Order Service requests reservation from Inventory Service
5. Inventory Service reserves stock (TTL: 15 min)
6. Order Service requests payment from Payment Service
7. Payment Service processes with external provider
8. Payment Service publishes success/failure event
9. Order Service updates status
10. Inventory Service confirms or releases reservation
11. Notification Service sends confirmation to client

### Search Flow

1. Client types search term
2. API Gateway receives request
3. Search Service checks cache in Redis
4. If cache miss, queries Elasticsearch
5. Results are cached and returned
6. Indexing events update indexes asynchronously

## Resilience Characteristics

### Circuit Breaker
All services implement circuit breaker for downstream calls, preventing cascade failures.

### Retry with Backoff
Operations that may temporarily fail use retry with exponential backoff.

### Graceful Degradation
Non-critical features can be disabled without impacting the main flow.

### Idempotency
Critical operations (payments, reservations) are idempotent to support safe retry.

## Scalability

### Horizontal
- All stateless services can scale horizontally
- HPA configured for auto-scaling based on CPU/memory
- Queue workers scale independently

### Vertical
- PostgreSQL can scale vertically (larger instances)
- Redis can scale horizontally (cluster mode)

## Security

- All communications are encrypted (TLS)
- Authentication via JWT with refresh tokens
- Secrets managed by AWS Secrets Manager
- Network policies isolate services in Kubernetes
- PCI-DSS compliance for payment data

## Related Links

### Components
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

### Architecture
- [Data Flow](data-flow.md)
- [Security Model](security-model.md)
- [Integration Patterns](integration-patterns.md)
