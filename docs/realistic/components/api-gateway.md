# API Gateway

## Description

The API Gateway is the centralized entry point for all external requests directed to TechCorp's microservices. This component acts as an intelligent reverse proxy, performing traffic routing, load balancing, and security policy enforcement.

The gateway implements essential functionalities such as rate limiting, request transformation, response aggregation, and circuit breaker. All external client calls must pass through this component before reaching internal services.

Beyond basic routing, the API Gateway integrates with the auth-service for JWT token validation, ensuring that only authenticated and authorized requests reach business services.

## Owners

- **Team:** Platform Engineering
- **Tech Lead:** Ricardo Mendes
- **Slack:** #platform-gateway

## Technology Stack

- Language: Go 1.21
- Framework: Kong Gateway (OSS)
- Database: PostgreSQL 15 (configurations)
- Cache: Redis 7 (rate limiting)
- Service Mesh: Istio 1.19

## Configuration

### Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `GATEWAY_PORT` | Gateway listening port | `8080` |
| `ADMIN_PORT` | Administrative API port | `8001` |
| `DATABASE_URL` | PostgreSQL connection string | - |
| `REDIS_URL` | Redis URL for rate limiting | - |
| `AUTH_SERVICE_URL` | Authentication service URL | - |
| `LOG_LEVEL` | Log level (debug, info, warn, error) | `info` |
| `RATE_LIMIT_REQUESTS` | Requests per minute per client | `1000` |
| `RATE_LIMIT_WINDOW` | Time window for rate limit (seconds) | `60` |

### Configuration File

```yaml
# /etc/kong/kong.conf
database: postgres
pg_host: postgres.internal
pg_port: 5432
pg_database: kong
proxy_listen: 0.0.0.0:8080
admin_listen: 127.0.0.1:8001
```

## How to Run Locally

```bash
# Clone the repository
git clone git@github.com:techcorp/api-gateway.git
cd api-gateway

# Start dependencies with Docker Compose
docker-compose up -d postgres redis

# Run migrations
kong migrations bootstrap

# Start the gateway
kong start

# Verify it's running
curl http://localhost:8001/status
```

### Local Route Configuration

```bash
# Add route to auth-service
curl -X POST http://localhost:8001/services \
  -d name=auth-service \
  -d url=http://localhost:3001

curl -X POST http://localhost:8001/services/auth-service/routes \
  -d paths[]=/auth
```

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/api-gateway
- **Prometheus Metrics:** https://prometheus.techcorp.internal/targets
- **Logs:** Sent to Elasticsearch via Fluentd

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `kong_http_requests_total` | Total requests | - |
| `kong_latency_ms` | Request latency | > 500ms |
| `kong_bandwidth_bytes` | Bandwidth consumed | - |
| `kong_upstream_target_health` | Upstream health | unhealthy |

### Configured Alerts

- **GatewayHighLatency:** P99 latency above 500ms for 5 minutes
- **GatewayErrorRate:** 5xx error rate above 1% for 5 minutes
- **GatewayUpstreamDown:** Upstream unresponsive for 1 minute

## Troubleshooting

### Issue: Requests returning 502 Bad Gateway

**Cause:** The destination service is unavailable or not responding in time.

**Solution:**
1. Verify the destination service is running
2. Check upstream logs: `kubectl logs -l app=<service-name>`
3. Verify timeout configuration in Kong

### Issue: Rate limit being applied incorrectly

**Cause:** Incorrect client identification configuration.

**Solution:**
1. Verify the `X-Consumer-ID` header is being sent
2. Check rate-limiting plugin configuration
3. Clear Redis cache: `redis-cli FLUSHDB`

### Issue: JWT tokens being rejected

**Cause:** Public key from auth-service outdated in the gateway.

**Solution:**
1. Update the public key: `kong reload`
2. Verify synchronization with auth-service
3. Check token expiration at jwt.io

## Related Links

- [Auth Service](auth-service.md) - Integrated authentication service
- [System Architecture Overview](../architecture/system-overview.md) - Gateway context in the system
- [Deploy Guide](../runbooks/deploy-guide.md) - How to deploy the gateway
- [Authentication API](../apis/auth-api.md) - Authentication endpoints
