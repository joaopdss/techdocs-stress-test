# Problemas de Performance

## Visão Geral

Este documento descreve problemas comuns de performance na plataforma TechCorp e suas soluções. Use este guia para diagnosticar e resolver issues de latência, throughput e uso de recursos.

## Diagnóstico Inicial

### Checklist de Investigação

1. **Quando começou?**
   - Após deploy recente?
   - Horário de pico de tráfego?
   - Mudança de configuração?

2. **Qual o escopo?**
   - Todos os usuários ou subconjunto?
   - Todos os endpoints ou específico?
   - Todos os serviços ou isolado?

3. **Métricas afetadas?**
   - Latência (P50, P95, P99)
   - Taxa de erros
   - Throughput
   - Uso de CPU/Memória

### Ferramentas de Diagnóstico

| Ferramenta | Uso | Link |
|------------|-----|------|
| Grafana | Métricas e dashboards | grafana.techcorp.internal |
| Kibana | Logs centralizados | kibana.techcorp.internal |
| Jaeger | Traces distribuídos | jaeger.techcorp.internal |
| pg_stat_statements | Queries PostgreSQL | Via psql |

## Problemas de Aplicação

### Alta Latência de API

**Sintomas:**
- P99 acima de 1 segundo
- Usuários reportam lentidão

**Investigação:**

1. **Identificar endpoint lento:**
```promql
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{service="api-gateway"}[5m])) by (le, path)
)
```

2. **Verificar traces:**
   - Acessar Jaeger
   - Buscar por traces lentos do endpoint
   - Identificar span com maior duração

3. **Verificar dependências:**
   - Latência do banco de dados
   - Latência de serviços downstream
   - Latência de cache

**Soluções comuns:**

| Causa | Solução |
|-------|---------|
| Query lenta no banco | Adicionar índices, otimizar query |
| N+1 queries | Usar eager loading, batch queries |
| Serialização lenta | Reduzir payload, usar cache |
| Serviço downstream lento | Cache, circuit breaker, fallback |

---

### Alto Uso de Memória

**Sintomas:**
- Pods sendo OOMKilled
- Memória próxima do limite

**Investigação:**

1. **Verificar uso por pod:**
```bash
kubectl top pods -n production --sort-by=memory
```

2. **Verificar histórico no Grafana:**
   - Dashboard: Service Resources
   - Métrica: `container_memory_usage_bytes`

3. **Analisar heap (Java):**
```bash
kubectl exec -it <pod> -n production -- jmap -heap 1
```

4. **Analisar memória (Python):**
```bash
kubectl exec -it <pod> -n production -- python -c "import tracemalloc; tracemalloc.start()"
```

**Soluções comuns:**

| Causa | Solução |
|-------|---------|
| Memory leak | Identificar e corrigir código |
| Cache sem limite | Configurar max size e TTL |
| Payload grande em memória | Usar streaming |
| Conexões não fechadas | Verificar connection pools |

---

### Alto Uso de CPU

**Sintomas:**
- CPU próximo de 100%
- Throttling de pods

**Investigação:**

1. **Verificar uso por pod:**
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

**Soluções comuns:**

| Causa | Solução |
|-------|---------|
| Loop infinito | Corrigir código |
| Regex complexa | Otimizar pattern |
| Serialização JSON | Usar biblioteca mais rápida |
| Cálculos repetidos | Cachear resultados |

---

## Problemas de Banco de Dados

### Queries Lentas

**Sintomas:**
- Latência alta em endpoints com acesso a banco
- Alertas de query duration

**Investigação:**

1. **Identificar queries lentas:**
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

2. **Analisar plano de execução:**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 'uuid' ORDER BY created_at DESC;
```

3. **Verificar índices:**
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

**Soluções comuns:**

| Causa | Solução |
|-------|---------|
| Falta de índice | `CREATE INDEX CONCURRENTLY` |
| Estatísticas desatualizadas | `ANALYZE <table>` |
| Seq scan em tabela grande | Adicionar índice ou particionar |
| Lock contention | Otimizar transações |

---

### Conexões Esgotadas

**Sintomas:**
- Erro "too many connections"
- Timeouts ao conectar

**Investigação:**

1. **Verificar conexões ativas:**
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

2. **Verificar conexões idle longas:**
```sql
SELECT pid, usename, application_name, state, query_start
FROM pg_stat_activity
WHERE state = 'idle'
  AND query_start < now() - interval '10 minutes';
```

**Soluções:**

| Causa | Solução |
|-------|---------|
| Connection leak | Corrigir código (fechar conexões) |
| Pool muito grande | Reduzir pool size por serviço |
| Muitos serviços | Usar PgBouncer |

---

## Problemas de Cache

### Baixo Hit Rate

**Sintomas:**
- Hit rate abaixo de 70%
- Alta latência mesmo com cache

**Investigação:**

1. **Verificar hit rate:**
```bash
redis-cli INFO stats | grep keyspace
```

2. **Verificar padrão de keys:**
```bash
redis-cli --scan --pattern "*" | head -100
```

**Soluções:**

| Causa | Solução |
|-------|---------|
| TTL muito curto | Aumentar TTL |
| Key pattern ruim | Revisar estratégia de cache |
| Cache frio após deploy | Implementar warm-up |

---

### Memória Redis Cheia

**Sintomas:**
- Evictions aumentando
- Erro "OOM command not allowed"

**Investigação:**

1. **Verificar uso de memória:**
```bash
redis-cli INFO memory
```

2. **Identificar keys grandes:**
```bash
redis-cli --bigkeys
```

**Soluções:**

| Causa | Solução |
|-------|---------|
| Keys sem TTL | Adicionar TTL a todas as keys |
| Valores muito grandes | Comprimir ou reduzir dados |
| Capacidade insuficiente | Escalar cluster |

---

## Problemas de Fila

### Backlog Crescente

**Sintomas:**
- Mensagens acumulando na fila
- Processamento atrasado

**Investigação:**

1. **Verificar tamanho das filas:**
```bash
rabbitmqctl list_queues name messages messages_ready
```

2. **Verificar consumidores:**
```bash
rabbitmqctl list_consumers
```

3. **Verificar rate de processamento:**
   - Grafana > RabbitMQ > Message Rates

**Soluções:**

| Causa | Solução |
|-------|---------|
| Poucos consumers | Escalar workers |
| Consumer lento | Otimizar processamento |
| Burst de mensagens | Implementar throttling |

---

## Problemas de Rede

### Alta Latência Entre Serviços

**Sintomas:**
- Latência alta em chamadas internas
- Timeouts frequentes

**Investigação:**

1. **Verificar latência de rede:**
```bash
kubectl exec -it <pod> -n production -- ping <service>
```

2. **Verificar DNS:**
```bash
kubectl exec -it <pod> -n production -- nslookup <service>
```

3. **Verificar Istio/Service Mesh:**
```bash
istioctl proxy-config clusters <pod> -n production
```

**Soluções:**

| Causa | Solução |
|-------|---------|
| DNS lento | Verificar CoreDNS |
| Network Policy bloqueando | Revisar policies |
| Node sobrecarregado | Balancear pods |

---

## Otimizações Recomendadas

### Para APIs

- Implementar cache em endpoints frequentes
- Usar paginação com cursor
- Comprimir responses (gzip)
- Usar connection pooling

### Para Banco de Dados

- Índices em colunas de WHERE/JOIN
- VACUUM regular
- Read replicas para queries
- Connection pooling (PgBouncer)

### Para Cache

- TTL apropriado para cada tipo de dado
- Estratégia de invalidação clara
- Monitorar hit rate
- Evitar thundering herd

### Para Filas

- Prefetch count adequado
- Processamento em batch
- Dead letter queue para erros
- Monitorar lag

## Links Relacionados

- [Erros Comuns](common-errors.md) - Diagnóstico de erros
- [Database PostgreSQL](../components/database-postgres.md) - Banco de dados
- [Cache Service](../components/cache-service.md) - Redis
- [Queue Service](../components/queue-service.md) - RabbitMQ
- [Monitoring Stack](../components/monitoring-stack.md) - Observabilidade
