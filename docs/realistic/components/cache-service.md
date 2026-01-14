# Cache Service

## Description

The Cache Service is TechCorp's in-memory storage layer, providing fast access to frequently accessed data and reducing load on databases and downstream services.

This component manages multiple Redis clusters configured for different use cases: user sessions, query cache, rate limiting, and processing queues. Each cluster has optimized configurations for its specific purpose.

The service also offers an abstraction API that implements common patterns such as cache-aside, write-through, and tag-based invalidation. This allows other services to use cache consistently without knowing implementation details.

## Owners

- **Team:** Platform Engineering
- **Tech Lead:** Thiago Santos
- **Slack:** #platform-cache

## Technology Stack

- Runtime: Redis 7.2
- Proxy: Redis Sentinel (high availability)
- Wrapper Language: Go 1.21
- Monitoring: Redis Exporter

## Available Clusters

| Cluster | Purpose | Eviction Policy | Default TTL |
|---------|---------|-----------------|-------------|
| `cache-sessions` | User sessions | noeviction | 7 days |
| `cache-queries` | Query cache | allkeys-lru | 1 hour |
| `cache-ratelimit` | Rate limiting | volatile-ttl | 1 minute |
| `cache-distributed-locks` | Distributed locks | noeviction | 30 seconds |

## Configuration

### Environment Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `REDIS_SESSIONS_URL` | Sessions cluster URL | - |
| `REDIS_QUERIES_URL` | Queries cluster URL | - |
| `REDIS_RATELIMIT_URL` | Rate limit cluster URL | - |
| `REDIS_LOCKS_URL` | Locks cluster URL | - |
| `CACHE_DEFAULT_TTL_SECONDS` | Default TTL for entries | `3600` |
| `CACHE_MAX_MEMORY_MB` | Maximum memory per instance | `2048` |
| `SENTINEL_MASTER_NAME` | Master name in Sentinel | `mymaster` |

### Redis Configuration

```conf
# redis.conf (queries cluster)
maxmemory 2gb
maxmemory-policy allkeys-lru
appendonly no
save ""
tcp-keepalive 300
timeout 0
```

## How to Run Locally

```bash
# Start local Redis with Docker
docker run -d --name redis-local -p 6379:6379 redis:7.2

# Connect via CLI
redis-cli -h localhost -p 6379

# Test basic operations
SET test "value"
GET test
TTL test
```

### Using the Abstraction API

```go
// Example of cache client usage
import "github.com/techcorp/cache-client"

client := cache.NewClient(cache.Config{
    Cluster: "queries",
    DefaultTTL: 1 * time.Hour,
})

// Cache-aside pattern
data, err := client.GetOrSet("user:123", func() (interface{}, error) {
    return userService.FindByID("123")
}, cache.WithTTL(30 * time.Minute))

// Tag-based invalidation
client.InvalidateByTag("user:123")
```

## Usage Patterns

### 1. Cache-Aside (Lazy Loading)

```
1. Application checks if data is in cache
2. If yes (cache hit), return data
3. If no (cache miss), fetch from database
4. Store in cache for next requests
```

### 2. Write-Through

```
1. Application writes to cache
2. Cache propagates write to database
3. Ensures consistency between cache and database
```

### 3. Tag-Based Invalidation

```go
// When saving a product, associate tags
cache.Set("product:123", data, cache.Tags("products", "category:electronics"))

// When updating category, invalidate all related
cache.InvalidateByTag("category:electronics")
```

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/cache-service
- **Prometheus Metrics:** https://prometheus.techcorp.internal/targets
- **Logs:** Redis logs sent to Elasticsearch

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `redis_connected_clients` | Connected clients | > 1000 |
| `redis_used_memory_bytes` | Memory used | > 80% |
| `redis_keyspace_hits_total` | Cache hits | - |
| `redis_keyspace_misses_total` | Cache misses | - |
| `redis_commands_duration_seconds` | Command latency | > 10ms |

### Configured Alerts

- **CacheMemoryHigh:** Memory usage above 80%
- **CacheHitRateLow:** Hit rate below 60% for 10 minutes
- **CacheConnectionsHigh:** More than 1000 simultaneous connections
- **CacheSentinelFailover:** Sentinel failover detected

## Troubleshooting

### Issue: Very low hit rate

**Cause:** TTL too short, poorly structured keys, or cold cache.

**Solution:**
1. Analyze access pattern: `redis-cli MONITOR`
2. Check TTL distribution: `redis-cli DEBUG OBJECT <key>`
3. Consider cache pre-warming during deploys

### Issue: Memory reaching limit

**Cause:** Keys not expiring or memory leak.

**Solution:**
1. Check keys without TTL: `redis-cli --scan --pattern '*' | head`
2. Analyze memory by prefix: `redis-cli MEMORY DOCTOR`
3. Force eviction if necessary: `redis-cli CONFIG SET maxmemory-policy allkeys-lru`

### Issue: High latency in operations

**Cause:** Blocking commands, slow network, or overloaded cluster.

**Solution:**
1. Identify slow commands: `redis-cli SLOWLOG GET 10`
2. Avoid O(n) commands like KEYS, use SCAN
3. Check network latency: `redis-cli --latency`

### Issue: Inconsistent data between cache and database

**Cause:** Invalidation failure or race condition.

**Solution:**
1. Check invalidation logs
2. Implement optimistic locking for updates
3. Temporarily reduce TTL to force refresh

## Best Practices

### Key Naming

```
{type}:{id}:{subtype}
user:123:profile
product:456:stock
order:789:items
```

### Serialization

- Use MessagePack for structured data (smaller than JSON)
- Use simple strings for atomic values
- Compress large data with LZ4

### TTL Guidelines

| Data Type | Recommended TTL |
|-----------|-----------------|
| User session | 7 days |
| API cache | 5 minutes |
| Reference data | 1 hour |
| Counters | 1 minute |
| Locks | 30 seconds |

## Related Links

- [Auth Service](auth-service.md) - Session storage
- [API Gateway](api-gateway.md) - Rate limiting
- [Search Service](search-service.md) - Query cache
- [Scaling Guide](../runbooks/scaling-guide.md) - Scale Redis clusters
