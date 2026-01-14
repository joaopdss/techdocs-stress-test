# Auth Service

## Description

The Auth Service is TechCorp's central authentication service, responsible for managing identities, issuing access tokens, and validating user and system credentials. This component implements OAuth 2.0 and OpenID Connect protocols, offering support for multiple authentication flows.

The service maintains the complete lifecycle of user sessions, from the initial authentication process to token revocation. It also manages two-factor authentication (2FA) via TOTP and integrates with external identity providers through SAML 2.0.

The auth-service architecture was designed for high availability, with support for multiple replicas and distributed session storage. All cryptographic operations use modern algorithms and keys are automatically rotated.

## Owners

- **Team:** Identity & Access Management
- **Tech Lead:** Camila Rodrigues
- **Slack:** #iam-auth

## Technology Stack

- Language: Java 21
- Framework: Spring Boot 3.2
- Database: PostgreSQL 15
- Cache: Redis 7 (sessions)
- Message Broker: RabbitMQ 3.12

## Configuration

### Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `AUTH_PORT` | Service HTTP port | `3001` |
| `DATABASE_URL` | PostgreSQL connection string | - |
| `REDIS_URL` | Redis URL for sessions | - |
| `JWT_PRIVATE_KEY` | RSA private key for signing tokens | - |
| `JWT_PUBLIC_KEY` | RSA public key for validating tokens | - |
| `JWT_EXPIRATION_MINUTES` | Access token expiration time | `30` |
| `REFRESH_TOKEN_DAYS` | Refresh token expiration time | `7` |
| `MFA_ISSUER` | Issuer name for TOTP apps | `TechCorp` |
| `RABBITMQ_URL` | RabbitMQ URL | - |

### Security Configuration

```yaml
# application.yml
security:
  jwt:
    algorithm: RS256
    issuer: https://auth.techcorp.com
    audience: techcorp-services
  password:
    encoder: bcrypt
    strength: 12
  mfa:
    enabled: true
    totp-window: 1
```

## How to Run Locally

```bash
# Clone the repository
git clone git@github.com:techcorp/auth-service.git
cd auth-service

# Start dependencies
docker-compose up -d postgres redis rabbitmq

# Generate JWT keys (development)
./scripts/generate-keys.sh

# Compile and run
./mvnw spring-boot:run -Dspring.profiles.active=local

# Verify service health
curl http://localhost:3001/actuator/health
```

### Test User

```bash
# Create test user
curl -X POST http://localhost:3001/admin/users \
  -H "Content-Type: application/json" \
  -d '{"email": "test@techcorp.com", "password": "Test@123"}'
```

## Supported Authentication Flows

### 1. Resource Owner Password Credentials

Used for first-party applications (web portal, mobile app).

```bash
curl -X POST http://localhost:3001/oauth/token \
  -d grant_type=password \
  -d username=user@techcorp.com \
  -d password=secret \
  -d client_id=web-portal
```

### 2. Client Credentials

Used for service-to-service communication.

```bash
curl -X POST http://localhost:3001/oauth/token \
  -d grant_type=client_credentials \
  -d client_id=order-service \
  -d client_secret=<secret>
```

### 3. Authorization Code (with PKCE)

Used for third-party applications and SPAs.

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/auth-service
- **Prometheus Metrics:** https://prometheus.techcorp.internal/targets
- **Logs:** Sent to Elasticsearch via Fluentd

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `auth_login_attempts_total` | Total authentication attempts | - |
| `auth_login_failures_total` | Authentication failures | > 100/min |
| `auth_tokens_issued_total` | Tokens issued | - |
| `auth_token_validation_latency_ms` | Validation latency | > 50ms |

### Configured Alerts

- **AuthHighFailureRate:** Authentication failure rate above 10% for 5 minutes
- **AuthServiceDown:** Service not responding to health check for 1 minute
- **AuthTokenValidationSlow:** Token validation above 100ms P99

## Troubleshooting

### Issue: User cannot authenticate

**Cause:** Account locked after multiple failed attempts.

**Solution:**
1. Check account status: `SELECT * FROM users WHERE email = '<email>'`
2. Unlock account if necessary: `UPDATE users SET locked = false WHERE email = '<email>'`
3. Check attempt logs: filter by `auth.login.failed`

### Issue: JWT token being rejected by other services

**Cause:** Public key not synchronized or token expired.

**Solution:**
1. Verify token hasn't expired at jwt.io
2. Confirm all services use the same public key
3. Force key update: endpoint `/jwks` returns the current key

### Issue: MFA not working

**Cause:** Server clock out of sync or incorrect configuration.

**Solution:**
1. Check server NTP synchronization
2. Temporarily increase `totp-window` for diagnosis
3. Regenerate TOTP seed for the user

## Published Events

The auth-service publishes events to RabbitMQ for auditing:

| Event | Exchange | Description |
|-------|----------|-------------|
| `user.authenticated` | `auth.events` | Successful authentication |
| `user.authentication.failed` | `auth.events` | Authentication failure |
| `user.locked` | `auth.events` | Account locked |
| `token.revoked` | `auth.events` | Token manually revoked |

## Related Links

- [Authentication API](../apis/auth-api.md) - Available endpoints
- [User Service](user-service.md) - User data management
- [API Gateway](api-gateway.md) - Token validation at the gateway
- [Security Model](../architecture/security-model.md) - Security architecture
