# Performance Issues

## Overview

This document describes common performance issues on the TechCorp platform and their solutions. Use this guide to diagnose and resolve latency, throughput, and resource usage issues.

## Initial Diagnosis

### Investigation Checklist

1. **When did it start?**
   - After recent deploy?
   - Peak traffic time?
   - Configuration change?

2. **What is the scope?**
   - All users or subset?
   - All endpoints or specific?
   - All services or isolated?

3. **Affected metrics?**
   - Latency (P50, P95, P99)
   - Error rate
   - Throughput
   - CPU/Memory usage

### Diagnostic Tools

| Tool | Use | Link |
|------|-----|------|
| Grafana | Metrics and dashboards | grafana.techcorp.internal |
| Kibana | Centralized logs | kibana.techcorp.internal |
| Jaeger | Distributed traces | jaeger.techcorp.internal |
| pg_stat_statements | PostgreSQL queries | Via psql |

## Application Issues

### High API Latency

**Symptoms:**
- P99 above 1 second
- Users reporting slowness

**Investigation:**

1. **Identify slow endpoint:**
```promql
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{service="api-gateway"}[5m])) by (le, path)
)
```

2. **Check traces:**
   - Access Jaeger
   - Search for slow traces of the endpoint
   - Identify span with longest duration

3. **Check dependencies:**
   - Database latency
   - Downstream service latency
   - Cache latency

**Common solutions:**

| Cause | Solution |
|-------|----------|
| Slow database query | Add indexes, optimize query |
| N+1 queries | Use eager loading, batch queries |
| Slow serialization | Reduce payload, use cache |
| Slow downstream service | Cache, circuit breaker, fallback |

---

### High Memory Usage

**Symptoms:**
- Pods being OOMKilled
- Memory near limit

**Investigation:**

1. **Check usage by pod:**
```bash
kubectl top pods -n production --sort-by=memory
```

2. **Check history in Grafana:**
   - Dashboard: Service Resources
   - Metric: `container_memory_usage_bytes`

3. **Analyze heap (Java):**
```bash
kubectl exec -it <pod> -n production -- jmap -heap 1
```

4. **Analyze memory (Python):**
```bash
kubectl exec -it <pod> -n production -- python -c "import tracemalloc; tracemalloc.start()"
```

**Common solutions:**

| Cause | Solution |
|-------|----------|
| Memory leak | Identify and fix code |
| Cache without limit | Configure max size and TTL |
| Large payload in memory | Use streaming |
| Unclosed connections | Check connection pools |

---

### High CPU Usage

**Symptoms:**
- CPU near 100%
- Pod throttling

**Investigation:**

1. **Check usage by pod:**
```bash
kubectl top pods -n production --sort-by=cpu
```

2. **Profiling (Java):**
```bash
kubectl exec -it <pod> -n production -- jstack 1 > thread_dump.txt
```

3. **Profiling (Python):**
```bash
kubectl exec -it <pod> -n production -- py-spy top --pid 1
```

**Common solutions:**

| Cause | Solution |
|-------|----------|
| Infinite loop | Fix code |
| Complex regex | Optimize pattern |
| JSON serialization | Use faster library |
| Repeated calculations | Cache results |

---

## Database Issues

### Slow Queries

**Symptoms:**
- High latency on endpoints with database access
- Query duration alerts

**Investigation:**

1. **Identify slow queries:**
```sql
SELECT
  query,
  calls,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round(total_exec_time::numeric / 1000, 2) AS total_s
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

2. **Analyze execution plan:**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 'uuid' ORDER BY created_at DESC;
```

3. **Check indexes:**
```sql
SELECT
  schemaname || '.' || relname AS table,
  indexrelname AS index,
  idx_scan AS scans,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan < 50
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Common solutions:**

| Cause | Solution |
|-------|----------|
| Missing index | `CREATE INDEX CONCURRENTLY` |
| Outdated statistics | `ANALYZE <table>` |
| Seq scan on large table | Add index or partition |
| Lock contention | Optimize transactions |

---

### Exhausted Connections

**Symptoms:**
- Error "too many connections"
- Connection timeouts

**Investigation:**

1. **Check active connections:**
```sql
SELECT
  application_name,
  count(*) AS connections,
  count(*) FILTER (WHERE state = 'active') AS active,
  count(*) FILTER (WHERE state = 'idle') AS idle
FROM pg_stat_activity
GROUP BY application_name
ORDER BY connections DESC;
```

2. **Check long idle connections:**
```sql
SELECT pid, usename, application_name, state, query_start
FROM pg_stat_activity
WHERE state = 'idle'
  AND query_start < now() - interval '10 minutes';
```

**Solutions:**

| Cause | Solution |
|-------|----------|
| Connection leak | Fix code (close connections) |
| Pool too large | Reduce pool size per service |
| Too many services | Use PgBouncer |

---

## Cache Issues

### Low Hit Rate

**Symptoms:**
- Hit rate below 70%
- High latency even with cache

**Investigation:**

1. **Check hit rate:**
```bash
redis-cli INFO stats | grep keyspace
```

2. **Check key pattern:**
```bash
redis-cli --scan --pattern "*" | head -100
```

**Solutions:**

| Cause | Solution |
|-------|----------|
| TTL too short | Increase TTL |
| Bad key pattern | Review cache strategy |
| Cold cache after deploy | Implement warm-up |

---

### Redis Memory Full

**Symptoms:**
- Evictions increasing
- Error "OOM command not allowed"

**Investigation:**

1. **Check memory usage:**
```bash
redis-cli INFO memory
```

2. **Identify large keys:**
```bash
redis-cli --bigkeys
```

**Solutions:**

| Cause | Solution |
|-------|----------|
| Keys without TTL | Add TTL to all keys |
| Values too large | Compress or reduce data |
| Insufficient capacity | Scale cluster |

---

## Queue Issues

### Growing Backlog

**Symptoms:**
- Messages accumulating in queue
- Delayed processing

**Investigation:**

1. **Check queue sizes:**
```bash
rabbitmqctl list_queues name messages messages_ready
```

2. **Check consumers:**
```bash
rabbitmqctl list_consumers
```

3. **Check processing rate:**
   - Grafana > RabbitMQ > Message Rates

**Solutions:**

| Cause | Solution |
|-------|----------|
| Few consumers | Scale workers |
| Slow consumer | Optimize processing |
| Message burst | Implement throttling |

---

## Network Issues

### High Latency Between Services

**Symptoms:**
- High latency on internal calls
- Frequent timeouts

**Investigation:**

1. **Check network latency:**
```bash
kubectl exec -it <pod> -n production -- ping <service>
```

2. **Check DNS:**
```bash
kubectl exec -it <pod> -n production -- nslookup <service>
```

3. **Check Istio/Service Mesh:**
```bash
istioctl proxy-config clusters <pod> -n production
```

**Solutions:**

| Cause | Solution |
|-------|----------|
| Slow DNS | Check CoreDNS |
| Network Policy blocking | Review policies |
| Overloaded node | Balance pods |

---

## Recommended Optimizations

### For APIs

- Implement cache on frequent endpoints
- Use cursor-based pagination
- Compress responses (gzip)
- Use connection pooling

### For Database

- Indexes on WHERE/JOIN columns
- Regular VACUUM
- Read replicas for queries
- Connection pooling (PgBouncer)

### For Cache

- Appropriate TTL for each data type
- Clear invalidation strategy
- Monitor hit rate
- Avoid thundering herd

### For Queues

- Adequate prefetch count
- Batch processing
- Dead letter queue for errors
- Monitor lag

## Related Links

- [Common Errors](common-errors.md) - Error diagnosis
- [Database PostgreSQL](../components/database-postgres.md) - Database
- [Cache Service](../components/cache-service.md) - Redis
- [Queue Service](../components/queue-service.md) - RabbitMQ
- [Monitoring Stack](../components/monitoring-stack.md) - Observability
