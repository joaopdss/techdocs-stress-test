# Modelo de Segurança

## Visão Geral

Este documento descreve a arquitetura de segurança da plataforma TechCorp, incluindo autenticação, autorização, criptografia e conformidade. A segurança é tratada como requisito fundamental em todas as camadas do sistema.

## Princípios de Segurança

1. **Defense in Depth** - Múltiplas camadas de proteção
2. **Least Privilege** - Acesso mínimo necessário
3. **Zero Trust** - Nunca confiar, sempre verificar
4. **Security by Design** - Segurança desde a concepção

## Autenticação

### Fluxo de Autenticação

```
┌──────────┐     ┌───────────┐     ┌──────────────┐
│  Cliente │────►│API Gateway│────►│ Auth Service │
└──────────┘     └─────┬─────┘     └──────┬───────┘
                       │                   │
                       │    Valida JWT     │
                       │◄──────────────────┤
                       │                   │
                       │    Token válido   │
                       ├───────────────────►
                       │    (route request)│
```

### Tokens JWT

**Estrutura do Token:**

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

**Características:**

| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| Algoritmo | RS256 | RSA com SHA-256 |
| Expiração Access Token | 30 min | Curto para minimizar impacto de vazamento |
| Expiração Refresh Token | 7 dias | Permite renovação sem re-autenticação |
| Rotação de Chaves | Mensal | Chaves rotacionam automaticamente |

### Multi-Factor Authentication (MFA)

**Métodos Suportados:**

| Método | Descrição | Recomendação |
|--------|-----------|--------------|
| TOTP | App authenticator | Recomendado |
| SMS | Código via SMS | Backup |
| Email | Código via e-mail | Backup |

**Fluxo MFA:**

1. Usuário submete credenciais
2. Auth Service valida credenciais
3. Se MFA habilitado, retorna `mfa_required: true`
4. Cliente solicita código MFA ao usuário
5. Cliente envia código ao Auth Service
6. Auth Service valida código
7. Auth Service emite tokens

### Proteção contra Ataques

| Ataque | Mitigação |
|--------|-----------|
| Brute Force | Rate limiting (5 tentativas/minuto) |
| Credential Stuffing | Detecção de anomalias |
| Session Hijacking | Vinculação a device fingerprint |
| Token Theft | Short-lived tokens + refresh rotation |

## Autorização

### Role-Based Access Control (RBAC)

**Roles Disponíveis:**

| Role | Descrição | Exemplo de Usuário |
|------|-----------|-------------------|
| `customer` | Cliente final | Usuário do app |
| `support` | Atendimento | Operador de suporte |
| `operations` | Operações | Analista de fulfillment |
| `finance` | Financeiro | Analista financeiro |
| `admin` | Administrador | Tech Lead |
| `service` | Serviço (M2M) | order-service |

**Matriz de Permissões:**

| Recurso | customer | support | operations | admin |
|---------|----------|---------|------------|-------|
| Ler próprio perfil | ✓ | ✓ | ✓ | ✓ |
| Ler qualquer perfil | - | ✓ | ✓ | ✓ |
| Criar pedido | ✓ | - | ✓ | ✓ |
| Cancelar pedido | próprio | qualquer | qualquer | qualquer |
| Estornar pagamento | - | - | ✓ | ✓ |
| Gerenciar usuários | - | - | - | ✓ |

### Attribute-Based Access Control (ABAC)

Para cenários mais complexos, usamos ABAC:

```javascript
// Exemplo: Usuário só pode ver pedidos do próprio tenant
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

## Criptografia

### Em Trânsito

| Protocolo | Versão | Uso |
|-----------|--------|-----|
| TLS | 1.3 | Todas as comunicações |
| mTLS | 1.3 | Service mesh (Istio) |

**Configuração TLS:**

```yaml
# Cipher suites permitidos
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_GCM_SHA256
```

### Em Repouso

| Dado | Método | Chave |
|------|--------|-------|
| Banco de dados | AES-256 | AWS KMS |
| Backups | AES-256 | AWS KMS |
| Secrets | AES-256 | AWS Secrets Manager |
| Logs | AES-256 | AWS KMS |

### Dados Sensíveis

| Tipo | Tratamento |
|------|------------|
| Senhas | bcrypt (cost=12) |
| Cartões | Tokenização (nunca armazenamos) |
| CPF | Criptografia AES-256 + máscara em logs |
| Tokens | SHA-256 para armazenamento |

## Conformidade

### PCI-DSS

O Payment Service é certificado PCI-DSS Level 1:

- Dados de cartão nunca tocam nossos servidores
- Tokenização via iframe do provedor
- Segregação de rede para ambiente de pagamentos
- Logs de auditoria completos
- Testes de penetração anuais

### LGPD

**Direitos Implementados:**

| Direito | Implementação |
|---------|---------------|
| Acesso | Exportação de dados via Admin |
| Retificação | Edição de perfil |
| Exclusão | Processo de account deletion |
| Portabilidade | Exportação em JSON |

**Retenção de Dados:**

| Tipo | Retenção | Base Legal |
|------|----------|------------|
| Dados de pedidos | 5 anos | Obrigação fiscal |
| Logs de acesso | 6 meses | Legítimo interesse |
| Dados de conta | Enquanto ativa | Execução de contrato |
| Dados de marketing | Até revogação | Consentimento |

## Segurança de Rede

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

### Segmentação de Rede

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

## Gestão de Secrets

### AWS Secrets Manager

Todos os secrets são armazenados no AWS Secrets Manager:

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

### Rotação de Secrets

| Secret | Frequência | Automático |
|--------|------------|------------|
| JWT Keys | Mensal | Sim |
| Database passwords | Trimestral | Sim |
| API Keys | Semestral | Manual |
| Service credentials | Trimestral | Sim |

## Auditoria

### Logs de Segurança

Todos os eventos de segurança são registrados:

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

### Eventos Auditados

| Evento | Severidade | Alerta |
|--------|------------|--------|
| Login success | Info | Não |
| Login failure | Warning | > 5/min |
| MFA failure | Warning | > 3/min |
| Account locked | High | Sim |
| Permission denied | Warning | > 10/min |
| Suspicious activity | Critical | Sim |

## Links Relacionados

- [Auth Service](../components/auth-service.md) - Serviço de autenticação
- [API de Autenticação](../apis/auth-api.md) - Endpoints de auth
- [API Gateway](../components/api-gateway.md) - Validação de tokens
- [Payment Service](../components/payment-service.md) - Segurança de pagamentos
