# User Service

## Description

The User Service is the microservice responsible for complete user lifecycle management on the TechCorp platform. This component centralizes all operations related to registration data, preferences, profiles, and user history.

The service implements complete CRUD operations for users, as well as advanced functionalities such as soft delete, change history, and duplicate account merging. It also manages organizational hierarchy, allowing users to be associated with teams, departments, and business units.

Integration with auth-service ensures that critical changes (such as email or phone) go through validation before being applied. The user-service also communicates with the notification-service to send registration change confirmations.

## Owners

- **Team:** Customer Platform
- **Tech Lead:** Fernando Costa
- **Slack:** #customer-users

## Technology Stack

- Language: Python 3.11
- Framework: FastAPI 0.104
- Database: PostgreSQL 15
- Cache: Redis 7
- ORM: SQLAlchemy 2.0

## Configuration

### Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `USER_SERVICE_PORT` | Service HTTP port | `3002` |
| `DATABASE_URL` | PostgreSQL connection string | - |
| `REDIS_URL` | Redis URL | - |
| `AUTH_SERVICE_URL` | auth-service URL | - |
| `NOTIFICATION_SERVICE_URL` | notification-service URL | - |
| `LOG_LEVEL` | Log level | `INFO` |
| `PAGINATION_DEFAULT_SIZE` | Default items per page | `20` |
| `PAGINATION_MAX_SIZE` | Maximum items per page | `100` |

### Database Configuration

```python
# config/database.py
SQLALCHEMY_DATABASE_URL = os.getenv("DATABASE_URL")
SQLALCHEMY_POOL_SIZE = 10
SQLALCHEMY_MAX_OVERFLOW = 20
SQLALCHEMY_POOL_TIMEOUT = 30
```

## How to Run Locally

```bash
# Clone the repository
git clone git@github.com:techcorp/user-service.git
cd user-service

# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Start dependencies
docker-compose up -d postgres redis

# Run migrations
alembic upgrade head

# Start the service
uvicorn main:app --reload --port 3002

# Access documentation
open http://localhost:3002/docs
```

## Data Model

### User Entity

```python
class User:
    id: UUID
    email: str
    phone: Optional[str]
    first_name: str
    last_name: str
    document_number: str  # Tax ID
    birth_date: date
    status: UserStatus  # ACTIVE, INACTIVE, PENDING, BLOCKED
    organization_id: UUID
    department_id: Optional[UUID]
    created_at: datetime
    updated_at: datetime
    deleted_at: Optional[datetime]
```

### UserPreferences Entity

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

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/user-service
- **Prometheus Metrics:** https://prometheus.techcorp.internal/targets
- **Logs:** Sent to Elasticsearch via Fluentd

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `user_service_requests_total` | Total requests | - |
| `user_service_request_duration_seconds` | Request latency | > 500ms |
| `user_service_db_connections` | Active connections in pool | > 80% |
| `user_service_cache_hit_ratio` | Cache hit ratio | < 70% |

### Configured Alerts

- **UserServiceHighLatency:** P99 latency above 500ms for 5 minutes
- **UserServiceDBConnectionsHigh:** Connection pool above 80%
- **UserServiceDown:** Service not responding to health check for 1 minute

## Troubleshooting

### Issue: Slow user searches

**Cause:** Missing indexes or cache not configured.

**Solution:**
1. Check if indexes exist on search columns: `email`, `document_number`
2. Check Redis cache hit ratio
3. Analyze slow queries in pg_stat_statements

### Issue: Error updating user email

**Cause:** Pending validation in auth-service or email already in use.

**Solution:**
1. Check if email is not registered: `SELECT * FROM users WHERE email = '<email>'`
2. Check for pending validation in auth-service
3. Check integration error logs

### Issue: Inconsistent data after account merge

**Cause:** Merge process interrupted.

**Solution:**
1. Check merge status in `user_merge_history` table
2. Execute rollback if necessary: `POST /admin/users/merge/<merge_id>/rollback`
3. Re-execute merge with `--force` flag

## Consumed Events

| Event | Source | Action |
|-------|--------|--------|
| `user.email.verified` | auth-service | Updates verification status |
| `user.phone.verified` | auth-service | Updates verification status |
| `order.completed` | order-service | Updates user metrics |

## Published Events

| Event | Exchange | Description |
|-------|----------|-------------|
| `user.created` | `user.events` | New user registered |
| `user.updated` | `user.events` | User data changed |
| `user.deleted` | `user.events` | User removed (soft delete) |
| `user.preferences.updated` | `user.events` | Preferences changed |

## Related Links

- [Users API](../apis/users-api.md) - Available routes
- [Auth Service](auth-service.md) - Authentication and validation
- [Notification Service](notification-service.md) - Notification sending
- [User Management](user-management.md) - Administration screen
- [Data Flow](../architecture/data-flow.md) - Service integration
