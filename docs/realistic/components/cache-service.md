# Cache Service

## Descrição

O Cache Service é a camada de armazenamento em memória da TechCorp, fornecendo acesso rápido a dados frequentemente acessados e reduzindo a carga nos bancos de dados e serviços downstream.

Este componente gerencia múltiplos clusters Redis configurados para diferentes casos de uso: sessões de usuário, cache de consultas, rate limiting e filas de processamento. Cada cluster possui configurações otimizadas para seu propósito específico.

O serviço também oferece uma API de abstração que implementa padrões comuns como cache-aside, write-through e invalidação por tags. Isso permite que outros serviços utilizem cache de forma consistente sem conhecer detalhes de implementação.

## Responsáveis

- **Time:** Platform Engineering
- **Tech Lead:** Thiago Santos
- **Slack:** #platform-cache

## Stack Tecnológica

- Runtime: Redis 7.2
- Proxy: Redis Sentinel (alta disponibilidade)
- Linguagem do wrapper: Go 1.21
- Monitoramento: Redis Exporter

## Clusters Disponíveis

| Cluster | Propósito | Política de Eviction | TTL Padrão |
|---------|-----------|---------------------|------------|
| `cache-sessions` | Sessões de usuário | noeviction | 7 dias |
| `cache-queries` | Cache de consultas | allkeys-lru | 1 hora |
| `cache-ratelimit` | Rate limiting | volatile-ttl | 1 minuto |
| `cache-distributed-locks` | Locks distribuídos | noeviction | 30 segundos |

## Configuração

### Variáveis de Ambiente

| Variável | Descrição | Valor Padrão |
|----------|-----------|--------------|
| `REDIS_SESSIONS_URL` | URL do cluster de sessões | - |
| `REDIS_QUERIES_URL` | URL do cluster de queries | - |
| `REDIS_RATELIMIT_URL` | URL do cluster de rate limit | - |
| `REDIS_LOCKS_URL` | URL do cluster de locks | - |
| `CACHE_DEFAULT_TTL_SECONDS` | TTL padrão para entradas | `3600` |
| `CACHE_MAX_MEMORY_MB` | Memória máxima por instância | `2048` |
| `SENTINEL_MASTER_NAME` | Nome do master no Sentinel | `mymaster` |

### Configuração do Redis

```conf
# redis.conf (cluster queries)
maxmemory 2gb
maxmemory-policy allkeys-lru
appendonly no
save ""
tcp-keepalive 300
timeout 0
```

## Como Executar Localmente

```bash
# Subir Redis local com Docker
docker run -d --name redis-local -p 6379:6379 redis:7.2

# Conectar via CLI
redis-cli -h localhost -p 6379

# Testar operações básicas
SET teste "valor"
GET teste
TTL teste
```

### Usando a API de Abstração

```go
// Exemplo de uso do cache client
import "github.com/techcorp/cache-client"

client := cache.NewClient(cache.Config{
    Cluster: "queries",
    DefaultTTL: 1 * time.Hour,
})

// Cache-aside pattern
data, err := client.GetOrSet("user:123", func() (interface{}, error) {
    return userService.FindByID("123")
}, cache.WithTTL(30 * time.Minute))

// Invalidação por tags
client.InvalidateByTag("user:123")
```

## Padrões de Uso

### 1. Cache-Aside (Lazy Loading)

```
1. Aplicação verifica se dado está no cache
2. Se sim (cache hit), retorna dado
3. Se não (cache miss), busca no banco
4. Armazena no cache para próximas requisições
```

### 2. Write-Through

```
1. Aplicação escreve no cache
2. Cache propaga escrita para banco
3. Garante consistência entre cache e banco
```

### 3. Invalidação por Tags

```go
// Ao salvar um produto, associar tags
cache.Set("product:123", data, cache.Tags("products", "category:eletronicos"))

// Ao atualizar categoria, invalidar todos relacionados
cache.InvalidateByTag("category:eletronicos")
```

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/cache-service
- **Métricas Prometheus:** https://prometheus.techcorp.internal/targets
- **Logs:** Logs do Redis enviados para Elasticsearch

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `redis_connected_clients` | Clientes conectados | > 1000 |
| `redis_used_memory_bytes` | Memória utilizada | > 80% |
| `redis_keyspace_hits_total` | Cache hits | - |
| `redis_keyspace_misses_total` | Cache misses | - |
| `redis_commands_duration_seconds` | Latência de comandos | > 10ms |

### Alertas Configurados

- **CacheMemoryHigh:** Uso de memória acima de 80%
- **CacheHitRateLow:** Taxa de hit abaixo de 60% por 10 minutos
- **CacheConnectionsHigh:** Mais de 1000 conexões simultâneas
- **CacheSentinelFailover:** Failover do Sentinel detectado

## Troubleshooting

### Problema: Taxa de hit muito baixa

**Causa:** TTL muito curto, chaves mal estruturadas ou cache frio.

**Solução:**
1. Analisar padrão de acesso: `redis-cli MONITOR`
2. Verificar distribuição de TTL: `redis-cli DEBUG OBJECT <key>`
3. Considerar pré-aquecimento do cache em deploys

### Problema: Memória chegando no limite

**Causa:** Chaves não expirando ou vazamento de memória.

**Solução:**
1. Verificar chaves sem TTL: `redis-cli --scan --pattern '*' | head`
2. Analisar memória por prefixo: `redis-cli MEMORY DOCTOR`
3. Forçar eviction se necessário: `redis-cli CONFIG SET maxmemory-policy allkeys-lru`

### Problema: Latência alta em operações

**Causa:** Comandos bloqueantes, rede lenta ou cluster sobrecarregado.

**Solução:**
1. Identificar comandos lentos: `redis-cli SLOWLOG GET 10`
2. Evitar comandos O(n) como KEYS, usar SCAN
3. Verificar latência de rede: `redis-cli --latency`

### Problema: Dados inconsistentes entre cache e banco

**Causa:** Falha na invalidação ou race condition.

**Solução:**
1. Verificar logs de invalidação
2. Implementar lock otimista para atualizações
3. Reduzir TTL temporariamente para forçar refresh

## Boas Práticas

### Nomenclatura de Chaves

```
{tipo}:{id}:{subtipo}
user:123:profile
product:456:stock
order:789:items
```

### Serialização

- Use MessagePack para dados estruturados (menor que JSON)
- Use strings simples para valores atômicos
- Comprima dados grandes com LZ4

### TTL Guidelines

| Tipo de Dado | TTL Recomendado |
|--------------|-----------------|
| Sessão de usuário | 7 dias |
| Cache de API | 5 minutos |
| Dados de referência | 1 hora |
| Contadores | 1 minuto |
| Locks | 30 segundos |

## Links Relacionados

- [Auth Service](auth-service.md) - Armazenamento de sessões
- [API Gateway](api-gateway.md) - Rate limiting
- [Search Service](search-service.md) - Cache de queries
- [Guia de Escalar](../runbooks/scaling-guide.md) - Escalar clusters Redis
