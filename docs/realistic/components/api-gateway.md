# API Gateway

## Descrição

O API Gateway é o ponto de entrada centralizado para todas as requisições externas direcionadas aos microsserviços da TechCorp. Este componente atua como um proxy reverso inteligente, realizando roteamento de tráfego, balanceamento de carga e aplicação de políticas de segurança.

O gateway implementa funcionalidades essenciais como rate limiting, transformação de requisições, agregação de respostas e circuit breaker. Todas as chamadas de clientes externos passam obrigatoriamente por este componente antes de alcançarem os serviços internos.

Além do roteamento básico, o API Gateway integra-se com o auth-service para validação de tokens JWT, garantindo que apenas requisições autenticadas e autorizadas alcancem os serviços de negócio.

## Responsáveis

- **Time:** Platform Engineering
- **Tech Lead:** Ricardo Mendes
- **Slack:** #platform-gateway

## Stack Tecnológica

- Linguagem: Go 1.21
- Framework: Kong Gateway (OSS)
- Banco de dados: PostgreSQL 15 (configurações)
- Cache: Redis 7 (rate limiting)
- Service Mesh: Istio 1.19

## Configuração

### Variáveis de Ambiente

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| `GATEWAY_PORT` | Porta de escuta do gateway | `8080` |
| `ADMIN_PORT` | Porta da API administrativa | `8001` |
| `DATABASE_URL` | Connection string do PostgreSQL | - |
| `REDIS_URL` | URL do Redis para rate limiting | - |
| `AUTH_SERVICE_URL` | URL do serviço de autenticação | - |
| `LOG_LEVEL` | Nível de log (debug, info, warn, error) | `info` |
| `RATE_LIMIT_REQUESTS` | Requisições por minuto por cliente | `1000` |
| `RATE_LIMIT_WINDOW` | Janela de tempo para rate limit (segundos) | `60` |

### Arquivo de Configuração

```yaml
# /etc/kong/kong.conf
database: postgres
pg_host: postgres.internal
pg_port: 5432
pg_database: kong
proxy_listen: 0.0.0.0:8080
admin_listen: 127.0.0.1:8001
```

## Como Executar Localmente

```bash
# Clonar o repositório
git clone git@github.com:techcorp/api-gateway.git
cd api-gateway

# Subir dependências com Docker Compose
docker-compose up -d postgres redis

# Executar migrações
kong migrations bootstrap

# Iniciar o gateway
kong start

# Verificar se está rodando
curl http://localhost:8001/status
```

### Configuração de Rotas Locais

```bash
# Adicionar rota para o auth-service
curl -X POST http://localhost:8001/services \
  -d name=auth-service \
  -d url=http://localhost:3001

curl -X POST http://localhost:8001/services/auth-service/routes \
  -d paths[]=/auth
```

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/api-gateway
- **Métricas Prometheus:** https://prometheus.techcorp.internal/targets
- **Logs:** Enviados para Elasticsearch via Fluentd

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `kong_http_requests_total` | Total de requisições | - |
| `kong_latency_ms` | Latência das requisições | > 500ms |
| `kong_bandwidth_bytes` | Bandwidth consumido | - |
| `kong_upstream_target_health` | Saúde dos upstreams | unhealthy |

### Alertas Configurados

- **GatewayHighLatency:** Latência P99 acima de 500ms por 5 minutos
- **GatewayErrorRate:** Taxa de erros 5xx acima de 1% por 5 minutos
- **GatewayUpstreamDown:** Upstream não responsivo por 1 minuto

## Troubleshooting

### Problema: Requisições retornando 502 Bad Gateway

**Causa:** O serviço de destino está indisponível ou não responde a tempo.

**Solução:**
1. Verificar se o serviço de destino está rodando
2. Checar logs do upstream: `kubectl logs -l app=<service-name>`
3. Verificar configuração de timeout no Kong

### Problema: Rate limit sendo aplicado incorretamente

**Causa:** Configuração de identificação do cliente incorreta.

**Solução:**
1. Verificar se o header `X-Consumer-ID` está sendo enviado
2. Checar configuração do plugin rate-limiting
3. Limpar cache do Redis: `redis-cli FLUSHDB`

### Problema: Tokens JWT sendo rejeitados

**Causa:** Chave pública do auth-service desatualizada no gateway.

**Solução:**
1. Atualizar a chave pública: `kong reload`
2. Verificar sincronização com auth-service
3. Checar expiração do token no jwt.io

## Links Relacionados

- [Auth Service](auth-service.md) - Serviço de autenticação integrado
- [Visão Geral da Arquitetura](../architecture/system-overview.md) - Contexto do gateway no sistema
- [Guia de Deploy](../runbooks/deploy-guide.md) - Como fazer deploy do gateway
- [API de Autenticação](../apis/auth-api.md) - Endpoints de autenticação
