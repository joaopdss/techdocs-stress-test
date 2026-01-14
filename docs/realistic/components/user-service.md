# User Service

## Descrição

O User Service é o microsserviço responsável pelo gerenciamento completo do ciclo de vida dos usuários na plataforma TechCorp. Este componente centraliza todas as operações relacionadas a dados cadastrais, preferências, perfis e histórico de usuários.

O serviço implementa operações CRUD completas para usuários, além de funcionalidades avançadas como soft delete, histórico de alterações e merge de contas duplicadas. Ele também gerencia a hierarquia organizacional, permitindo associar usuários a times, departamentos e unidades de negócio.

A integração com o auth-service garante que alterações críticas (como e-mail ou telefone) passem por validação antes de serem efetivadas. O user-service também se comunica com o notification-service para enviar confirmações de alterações cadastrais.

## Responsáveis

- **Time:** Customer Platform
- **Tech Lead:** Fernando Costa
- **Slack:** #customer-users

## Stack Tecnológica

- Linguagem: Python 3.11
- Framework: FastAPI 0.104
- Banco de dados: PostgreSQL 15
- Cache: Redis 7
- ORM: SQLAlchemy 2.0

## Configuração

### Variáveis de Ambiente

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| `USER_SERVICE_PORT` | Porta HTTP do serviço | `3002` |
| `DATABASE_URL` | Connection string PostgreSQL | - |
| `REDIS_URL` | URL do Redis | - |
| `AUTH_SERVICE_URL` | URL do auth-service | - |
| `NOTIFICATION_SERVICE_URL` | URL do notification-service | - |
| `LOG_LEVEL` | Nível de log | `INFO` |
| `PAGINATION_DEFAULT_SIZE` | Itens por página padrão | `20` |
| `PAGINATION_MAX_SIZE` | Máximo de itens por página | `100` |

### Configuração de Database

```python
# config/database.py
SQLALCHEMY_DATABASE_URL = os.getenv("DATABASE_URL")
SQLALCHEMY_POOL_SIZE = 10
SQLALCHEMY_MAX_OVERFLOW = 20
SQLALCHEMY_POOL_TIMEOUT = 30
```

## Como Executar Localmente

```bash
# Clonar o repositório
git clone git@github.com:techcorp/user-service.git
cd user-service

# Criar ambiente virtual
python -m venv venv
source venv/bin/activate

# Instalar dependências
pip install -r requirements.txt

# Subir dependências
docker-compose up -d postgres redis

# Executar migrações
alembic upgrade head

# Iniciar o serviço
uvicorn main:app --reload --port 3002

# Acessar documentação
open http://localhost:3002/docs
```

## Modelo de Dados

### Entidade User

```python
class User:
    id: UUID
    email: str
    phone: Optional[str]
    first_name: str
    last_name: str
    document_number: str  # CPF
    birth_date: date
    status: UserStatus  # ACTIVE, INACTIVE, PENDING, BLOCKED
    organization_id: UUID
    department_id: Optional[UUID]
    created_at: datetime
    updated_at: datetime
    deleted_at: Optional[datetime]
```

### Entidade UserPreferences

```python
class UserPreferences:
    user_id: UUID
    language: str  # pt-BR, en-US, es-ES
    timezone: str
    notification_email: bool
    notification_sms: bool
    notification_push: bool
    theme: str  # light, dark, system
```

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/user-service
- **Métricas Prometheus:** https://prometheus.techcorp.internal/targets
- **Logs:** Enviados para Elasticsearch via Fluentd

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `user_service_requests_total` | Total de requisições | - |
| `user_service_request_duration_seconds` | Latência das requisições | > 500ms |
| `user_service_db_connections` | Conexões ativas no pool | > 80% |
| `user_service_cache_hit_ratio` | Taxa de acerto do cache | < 70% |

### Alertas Configurados

- **UserServiceHighLatency:** Latência P99 acima de 500ms por 5 minutos
- **UserServiceDBConnectionsHigh:** Pool de conexões acima de 80%
- **UserServiceDown:** Serviço não responde health check por 1 minuto

## Troubleshooting

### Problema: Lentidão em buscas de usuários

**Causa:** Falta de índices ou cache não configurado.

**Solução:**
1. Verificar se há índices nas colunas de busca: `email`, `document_number`
2. Verificar hit ratio do cache Redis
3. Analisar queries lentas no pg_stat_statements

### Problema: Erro ao atualizar e-mail do usuário

**Causa:** Validação pendente no auth-service ou e-mail já em uso.

**Solução:**
1. Verificar se e-mail não está cadastrado: `SELECT * FROM users WHERE email = '<email>'`
2. Checar se há validação pendente no auth-service
3. Verificar logs de erro da integração

### Problema: Dados inconsistentes após merge de contas

**Causa:** Processo de merge interrompido.

**Solução:**
1. Verificar status do merge na tabela `user_merge_history`
2. Executar rollback se necessário: `POST /admin/users/merge/<merge_id>/rollback`
3. Reexecutar merge com flag `--force`

## Eventos Consumidos

| Evento | Origem | Ação |
|--------|--------|------|
| `user.email.verified` | auth-service | Atualiza status de verificação |
| `user.phone.verified` | auth-service | Atualiza status de verificação |
| `order.completed` | order-service | Atualiza métricas do usuário |

## Eventos Publicados

| Evento | Exchange | Descrição |
|--------|----------|-----------|
| `user.created` | `user.events` | Novo usuário cadastrado |
| `user.updated` | `user.events` | Dados do usuário alterados |
| `user.deleted` | `user.events` | Usuário removido (soft delete) |
| `user.preferences.updated` | `user.events` | Preferências alteradas |

## Links Relacionados

- [API de Usuários](../apis/users-api.md) - Rotas disponíveis
- [Auth Service](auth-service.md) - Autenticação e validação
- [Notification Service](notification-service.md) - Envio de notificações
- [User Management](user-management.md) - Tela de administração
- [Fluxo de Dados](../architecture/data-flow.md) - Integração entre serviços
