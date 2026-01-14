# Auth Service

## Descrição

O Auth Service é o serviço central de autenticação da TechCorp, responsável por gerenciar identidades, emitir tokens de acesso e validar credenciais de usuários e sistemas. Este componente implementa os protocolos OAuth 2.0 e OpenID Connect, oferecendo suporte a múltiplos fluxos de autenticação.

O serviço mantém o ciclo de vida completo das sessões de usuário, desde o processo de autenticação inicial até a revogação de tokens. Ele também gerencia autenticação de dois fatores (2FA) via TOTP e integra-se com provedores de identidade externos através de SAML 2.0.

A arquitetura do auth-service foi projetada para alta disponibilidade, com suporte a múltiplas réplicas e armazenamento distribuído de sessões. Todas as operações criptográficas utilizam algoritmos modernos e as chaves são rotacionadas automaticamente.

## Responsáveis

- **Time:** Identity & Access Management
- **Tech Lead:** Camila Rodrigues
- **Slack:** #iam-auth

## Stack Tecnológica

- Linguagem: Java 21
- Framework: Spring Boot 3.2
- Banco de dados: PostgreSQL 15
- Cache: Redis 7 (sessões)
- Message Broker: RabbitMQ 3.12

## Configuração

### Variáveis de Ambiente

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| `AUTH_PORT` | Porta HTTP do serviço | `3001` |
| `DATABASE_URL` | Connection string PostgreSQL | - |
| `REDIS_URL` | URL do Redis para sessões | - |
| `JWT_PRIVATE_KEY` | Chave privada RSA para assinar tokens | - |
| `JWT_PUBLIC_KEY` | Chave pública RSA para validar tokens | - |
| `JWT_EXPIRATION_MINUTES` | Tempo de expiração do access token | `30` |
| `REFRESH_TOKEN_DAYS` | Tempo de expiração do refresh token | `7` |
| `MFA_ISSUER` | Nome do emissor para apps TOTP | `TechCorp` |
| `RABBITMQ_URL` | URL do RabbitMQ | - |

### Configuração de Segurança

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

## Como Executar Localmente

```bash
# Clonar o repositório
git clone git@github.com:techcorp/auth-service.git
cd auth-service

# Subir dependências
docker-compose up -d postgres redis rabbitmq

# Gerar chaves JWT (desenvolvimento)
./scripts/generate-keys.sh

# Compilar e executar
./mvnw spring-boot:run -Dspring.profiles.active=local

# Verificar saúde do serviço
curl http://localhost:3001/actuator/health
```

### Usuário de Teste

```bash
# Criar usuário de teste
curl -X POST http://localhost:3001/admin/users \
  -H "Content-Type: application/json" \
  -d '{"email": "teste@techcorp.com", "password": "Test@123"}'
```

## Fluxos de Autenticação Suportados

### 1. Resource Owner Password Credentials

Usado para aplicações first-party (web portal, mobile app).

```bash
curl -X POST http://localhost:3001/oauth/token \
  -d grant_type=password \
  -d username=user@techcorp.com \
  -d password=secret \
  -d client_id=web-portal
```

### 2. Client Credentials

Usado para comunicação entre serviços (service-to-service).

```bash
curl -X POST http://localhost:3001/oauth/token \
  -d grant_type=client_credentials \
  -d client_id=order-service \
  -d client_secret=<secret>
```

### 3. Authorization Code (com PKCE)

Usado para aplicações third-party e SPAs.

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/auth-service
- **Métricas Prometheus:** https://prometheus.techcorp.internal/targets
- **Logs:** Enviados para Elasticsearch via Fluentd

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `auth_login_attempts_total` | Total de tentativas de autenticação | - |
| `auth_login_failures_total` | Falhas de autenticação | > 100/min |
| `auth_tokens_issued_total` | Tokens emitidos | - |
| `auth_token_validation_latency_ms` | Latência de validação | > 50ms |

### Alertas Configurados

- **AuthHighFailureRate:** Taxa de falhas de autenticação acima de 10% por 5 minutos
- **AuthServiceDown:** Serviço não responde health check por 1 minuto
- **AuthTokenValidationSlow:** Validação de token acima de 100ms P99

## Troubleshooting

### Problema: Usuário não consegue realizar autenticação

**Causa:** Conta bloqueada após múltiplas tentativas falhas.

**Solução:**
1. Verificar status da conta: `SELECT * FROM users WHERE email = '<email>'`
2. Desbloquear conta se necessário: `UPDATE users SET locked = false WHERE email = '<email>'`
3. Verificar logs de tentativas: filtrar por `auth.login.failed`

### Problema: Token JWT sendo rejeitado por outros serviços

**Causa:** Chave pública não sincronizada ou token expirado.

**Solução:**
1. Verificar se o token não expirou em jwt.io
2. Confirmar que todos os serviços usam a mesma chave pública
3. Forçar atualização da chave: endpoint `/jwks` retorna a chave atual

### Problema: MFA não funcionando

**Causa:** Relógio do servidor dessincronizado ou configuração incorreta.

**Solução:**
1. Verificar sincronização NTP do servidor
2. Aumentar `totp-window` temporariamente para diagnóstico
3. Regenerar seed do TOTP para o usuário

## Eventos Publicados

O auth-service publica eventos no RabbitMQ para auditoria:

| Evento | Exchange | Descrição |
|--------|----------|-----------|
| `user.authenticated` | `auth.events` | Autenticação bem-sucedida |
| `user.authentication.failed` | `auth.events` | Falha na autenticação |
| `user.locked` | `auth.events` | Conta bloqueada |
| `token.revoked` | `auth.events` | Token revogado manualmente |

## Links Relacionados

- [API de Autenticação](../apis/auth-api.md) - Endpoints disponíveis
- [User Service](user-service.md) - Gerenciamento de dados de usuário
- [API Gateway](api-gateway.md) - Validação de tokens no gateway
- [Modelo de Segurança](../architecture/security-model.md) - Arquitetura de segurança
