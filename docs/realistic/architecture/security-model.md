# Security Model

## Overview

This document describes the security architecture of the TechCorp platform, including authentication, authorization, encryption, and compliance. Security is treated as a fundamental requirement across all system layers.

## Security Principles

1. **Defense in Depth** - Multiple layers of protection
2. **Least Privilege** - Minimum necessary access
3. **Zero Trust** - Never trust, always verify
4. **Security by Design** - Security from conception

## Authentication

### Authentication Flow

```
┌──────────┐     ┌───────────┐     ┌──────────────┐
│  Client  │────►│API Gateway│────►│ Auth Service │
└──────────┘     └─────┬─────┘     └──────┬───────┘
                       │                   │
                       │    Validates JWT  │
                       │◄──────────────────┤
                       │                   │
                       │    Token valid    │
                       ├───────────────────►
                       │    (route request)│
```

### JWT Tokens

**Token Structure:**

```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key-2024-01"
  },
  "payload": {
    "sub": "user_uuid",
    "email": "user@example.com",
    "roles": ["customer"],
    "permissions": ["read:profile", "write:orders"],
    "iat": 1705312200,
    "exp": 1705314000,
    "iss": "https://auth.techcorp.com",
    "aud": ["techcorp-api"]
  }
}
```

**Characteristics:**

| Parameter | Value | Description |
|-----------|-------|-------------|
| Algorithm | RS256 | RSA with SHA-256 |
| Access Token Expiration | 30 min | Short to minimize leak impact |
| Refresh Token Expiration | 7 days | Allows renewal without re-authentication |
| Key Rotation | Monthly | Keys rotate automatically |

### Multi-Factor Authentication (MFA)

**Supported Methods:**

| Method | Description | Recommendation |
|--------|-------------|----------------|
| TOTP | Authenticator app | Recommended |
| SMS | Code via SMS | Backup |
| Email | Code via email | Backup |

**MFA Flow:**

1. User submits credentials
2. Auth Service validates credentials
3. If MFA enabled, returns `mfa_required: true`
4. Client requests MFA code from user
5. Client sends code to Auth Service
6. Auth Service validates code
7. Auth Service issues tokens

### Attack Protection

| Attack | Mitigation |
|--------|------------|
| Brute Force | Rate limiting (5 attempts/minute) |
| Credential Stuffing | Anomaly detection |
| Session Hijacking | Device fingerprint binding |
| Token Theft | Short-lived tokens + refresh rotation |

## Authorization

### Role-Based Access Control (RBAC)

**Available Roles:**

| Role | Description | Example User |
|------|-------------|--------------|
| `customer` | End customer | App user |
| `support` | Customer service | Support operator |
| `operations` | Operations | Fulfillment analyst |
| `finance` | Finance | Financial analyst |
| `admin` | Administrator | Tech Lead |
| `service` | Service (M2M) | order-service |

**Permissions Matrix:**

| Resource | customer | support | operations | admin |
|----------|----------|---------|------------|-------|
| Read own profile | ✓ | ✓ | ✓ | ✓ |
| Read any profile | - | ✓ | ✓ | ✓ |
| Create order | ✓ | - | ✓ | ✓ |
| Cancel order | own | any | any | any |
| Refund payment | - | - | ✓ | ✓ |
| Manage users | - | - | - | ✓ |

### Attribute-Based Access Control (ABAC)

For more complex scenarios, we use ABAC:

```javascript
// Example: User can only see orders from their own tenant
{
  "resource": "orders",
  "action": "read",
  "conditions": {
    "resource.tenant_id": "${user.tenant_id}"
  }
}
```

### Service-to-Service Authentication

**Client Credentials Flow:**

```bash
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=order-service
&client_secret=<secret>
&scope=payments:write inventory:write
```

**Service Accounts:**

| Service | Permissions |
|---------|-------------|
| order-service | inventory:*, payments:*, notifications:write |
| payment-service | orders:write |
| notification-service | users:read |

## Encryption

### In Transit

| Protocol | Version | Usage |
|----------|---------|-------|
| TLS | 1.3 | All communications |
| mTLS | 1.3 | Service mesh (Istio) |

**TLS Configuration:**

```yaml
# Allowed cipher suites
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_GCM_SHA256
```

### At Rest

| Data | Method | Key |
|------|--------|-----|
| Database | AES-256 | AWS KMS |
| Backups | AES-256 | AWS KMS |
| Secrets | AES-256 | AWS Secrets Manager |
| Logs | AES-256 | AWS KMS |

### Sensitive Data

| Type | Treatment |
|------|-----------|
| Passwords | bcrypt (cost=12) |
| Cards | Tokenization (never stored) |
| SSN/Tax ID | AES-256 encryption + log masking |
| Tokens | SHA-256 for storage |

## Compliance

### PCI-DSS

The Payment Service is PCI-DSS Level 1 certified:

- Card data never touches our servers
- Tokenization via provider iframe
- Network segregation for payment environment
- Complete audit logs
- Annual penetration testing

### LGPD (Brazilian Data Protection Law)

**Implemented Rights:**

| Right | Implementation |
|-------|----------------|
| Access | Data export via Admin |
| Rectification | Profile editing |
| Deletion | Account deletion process |
| Portability | JSON export |

**Data Retention:**

| Type | Retention | Legal Basis |
|------|-----------|-------------|
| Order data | 5 years | Tax obligation |
| Access logs | 6 months | Legitimate interest |
| Account data | While active | Contract execution |
| Marketing data | Until revocation | Consent |

## Network Security

### Network Policies (Kubernetes)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: payment-service
        - podSelector:
            matchLabels:
              app: inventory-service
```

### Network Segmentation

```
Internet
    │
    ▼
┌───────────────┐
│   WAF/CDN     │  (Cloudflare)
└───────┬───────┘
        │
┌───────▼───────┐
│   Public LB   │  (ALB)
└───────┬───────┘
        │
┌───────▼───────────────────────────────────┐
│            VPC (10.0.0.0/16)              │
│  ┌─────────────────────────────────────┐  │
│  │        Public Subnet                │  │
│  │        (API Gateway only)           │  │
│  └─────────────────────────────────────┘  │
│  ┌─────────────────────────────────────┐  │
│  │        Private Subnet               │  │
│  │        (Microservices)              │  │
│  └─────────────────────────────────────┘  │
│  ┌─────────────────────────────────────┐  │
│  │        Data Subnet                  │  │
│  │        (RDS, Redis, RabbitMQ)       │  │
│  └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
```

## Secrets Management

### AWS Secrets Manager

All secrets are stored in AWS Secrets Manager:

```yaml
# External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: auth-service-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: auth-service-secrets
  data:
    - secretKey: jwt-private-key
      remoteRef:
        key: production/auth-service
        property: jwt_private_key
```

### Secrets Rotation

| Secret | Frequency | Automatic |
|--------|-----------|-----------|
| JWT Keys | Monthly | Yes |
| Database passwords | Quarterly | Yes |
| API Keys | Semi-annually | Manual |
| Service credentials | Quarterly | Yes |

## Auditing

### Security Logs

All security events are logged:

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "event_type": "authentication.success",
  "user_id": "uuid",
  "ip_address": "192.168.1.1",
  "user_agent": "Mozilla/5.0...",
  "device_fingerprint": "xxx",
  "mfa_used": true,
  "session_id": "xxx"
}
```

### Audited Events

| Event | Severity | Alert |
|-------|----------|-------|
| Login success | Info | No |
| Login failure | Warning | > 5/min |
| MFA failure | Warning | > 3/min |
| Account locked | High | Yes |
| Permission denied | Warning | > 10/min |
| Suspicious activity | Critical | Yes |

## Related Links

- [Auth Service](../components/auth-service.md) - Authentication service
- [Auth API](../apis/auth-api.md) - Auth endpoints
- [API Gateway](../components/api-gateway.md) - Token validation
- [Payment Service](../components/payment-service.md) - Payment security
