# Manutenção de Banco de Dados

## Visão Geral

Este guia descreve os procedimentos de manutenção do PostgreSQL na TechCorp. A manutenção regular é essencial para garantir performance, integridade dos dados e disponibilidade do sistema.

## Tarefas de Manutenção

### Manutenção Automática

| Tarefa | Frequência | Horário | Impacto |
|--------|------------|---------|---------|
| VACUUM automático | Contínuo | - | Baixo |
| Backup incremental | A cada hora | - | Nenhum |
| Backup completo | Diário | 03:00 UTC | Baixo |
| Atualização de estatísticas | Contínuo | - | Nenhum |

### Manutenção Manual

| Tarefa | Frequência | Janela | Impacto |
|--------|------------|--------|---------|
| VACUUM FULL | Trimestral | Madrugada | Alto |
| REINDEX | Mensal | Madrugada | Médio |
| Análise de queries lentas | Semanal | - | Nenhum |
| Limpeza de dados antigos | Mensal | Madrugada | Baixo |

## Procedimentos

### Verificar Saúde do Banco

```sql
-- Verificar bloat de tabelas
SELECT
  schemaname || '.' || tablename AS table,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
  pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_size,
  pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) AS index_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;

-- Verificar índices não utilizados
SELECT
  schemaname || '.' || relname AS table,
  indexrelname AS index,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size,
  idx_scan AS scans
FROM pg_stat_user_indexes
WHERE idx_scan < 50
ORDER BY pg_relation_size(indexrelid) DESC;

-- Verificar locks ativos
SELECT
  pid,
  usename,
  pg_blocking_pids(pid) AS blocked_by,
  query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### VACUUM

#### VACUUM Regular

Executado automaticamente pelo autovacuum. Para forçar:

```sql
-- VACUUM em tabela específica
VACUUM VERBOSE orders.orders;

-- VACUUM em todo o schema
DO $$
DECLARE
  tbl text;
BEGIN
  FOR tbl IN SELECT tablename FROM pg_tables WHERE schemaname = 'orders'
  LOOP
    EXECUTE 'VACUUM VERBOSE orders.' || quote_ident(tbl);
  END LOOP;
END;
$$;
```

#### VACUUM FULL

**ATENÇÃO:** VACUUM FULL bloqueia a tabela. Agendar em janela de manutenção.

```sql
-- Verificar necessidade
SELECT
  schemaname || '.' || relname AS table,
  n_dead_tup AS dead_tuples,
  pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_stat_user_tables
WHERE n_dead_tup > 100000
ORDER BY n_dead_tup DESC;

-- Executar VACUUM FULL
VACUUM FULL VERBOSE orders.orders;
```

### REINDEX

```sql
-- REINDEX de índice específico
REINDEX INDEX CONCURRENTLY orders.orders_user_id_idx;

-- REINDEX de tabela (todos os índices)
REINDEX TABLE CONCURRENTLY orders.orders;

-- REINDEX de todo o schema
REINDEX SCHEMA CONCURRENTLY orders;
```

**NOTA:** Use `CONCURRENTLY` para evitar bloqueio em produção.

### Análise de Queries Lentas

#### Habilitar pg_stat_statements

```sql
-- Verificar se extensão está ativa
SELECT * FROM pg_extension WHERE extname = 'pg_stat_statements';

-- Top 10 queries por tempo total
SELECT
  substring(query, 1, 100) AS query,
  calls,
  round(total_exec_time::numeric, 2) AS total_time_ms,
  round(mean_exec_time::numeric, 2) AS mean_time_ms,
  rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Queries com alta variância (instáveis)
SELECT
  substring(query, 1, 100) AS query,
  calls,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round(stddev_exec_time::numeric, 2) AS stddev_ms
FROM pg_stat_statements
WHERE calls > 100
ORDER BY stddev_exec_time DESC
LIMIT 10;
```

#### Análise de Query Específica

```sql
-- Plano de execução
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders.orders WHERE user_id = 'uuid' ORDER BY created_at DESC LIMIT 10;
```

### Limpeza de Dados Antigos

#### Configuração de Retenção

| Schema | Tabela | Retenção | Critério |
|--------|--------|----------|----------|
| auth | sessions | 30 dias | expired_at |
| orders | order_events | 1 ano | created_at |
| inventory | reservations | 7 dias | expires_at |
| audit | logs | 2 anos | created_at |

#### Script de Limpeza

```sql
-- Limpar sessões expiradas
DELETE FROM auth.sessions
WHERE expired_at < NOW() - INTERVAL '30 days';

-- Limpar reservas expiradas
DELETE FROM inventory.reservations
WHERE status = 'EXPIRED' AND expires_at < NOW() - INTERVAL '7 days';

-- Limpar eventos de pedidos antigos (em batches)
DO $$
DECLARE
  deleted_count INT;
BEGIN
  LOOP
    DELETE FROM orders.order_events
    WHERE id IN (
      SELECT id FROM orders.order_events
      WHERE created_at < NOW() - INTERVAL '1 year'
      LIMIT 10000
    );

    GET DIAGNOSTICS deleted_count = ROW_COUNT;
    RAISE NOTICE 'Deleted % rows', deleted_count;

    IF deleted_count = 0 THEN
      EXIT;
    END IF;

    -- Pausa entre batches para não sobrecarregar
    PERFORM pg_sleep(1);
  END LOOP;
END;
$$;
```

### Backup e Recovery

#### Verificar Backups

```bash
# Listar snapshots disponíveis
aws rds describe-db-snapshots \
  --db-instance-identifier techcorp-primary \
  --query 'DBSnapshots[*].[DBSnapshotIdentifier,SnapshotCreateTime,Status]' \
  --output table

# Verificar PITR
aws rds describe-db-instances \
  --db-instance-identifier techcorp-primary \
  --query 'DBInstances[*].LatestRestorableTime'
```

#### Restore de Backup

```bash
# Restore point-in-time (cria nova instância)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier techcorp-primary \
  --target-db-instance-identifier techcorp-restored \
  --restore-time "2024-01-15T10:30:00Z"

# Restore de snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier techcorp-restored \
  --db-snapshot-identifier techcorp-daily-20240115
```

#### Export de Tabela Específica

```bash
# Via pg_dump
pg_dump -h techcorp-replica.xxx.rds.amazonaws.com \
  -U app_user -d techcorp \
  -t orders.orders \
  -f orders_backup.sql

# Importar
psql -h techcorp-primary.xxx.rds.amazonaws.com \
  -U app_user -d techcorp \
  -f orders_backup.sql
```

### Monitoramento

#### Alertas de Manutenção

| Condição | Threshold | Ação |
|----------|-----------|------|
| Dead tuples > 10M | Warning | Agendar VACUUM |
| Bloat > 30% | Warning | Agendar VACUUM FULL |
| Conexões > 80% | Critical | Investigar |
| Replica lag > 60s | Critical | Investigar |

#### Queries para Dashboard

```sql
-- Conexões ativas por aplicação
SELECT
  application_name,
  count(*) AS connections,
  count(*) FILTER (WHERE state = 'active') AS active,
  count(*) FILTER (WHERE state = 'idle') AS idle
FROM pg_stat_activity
WHERE datname = 'techcorp'
GROUP BY application_name;

-- Transações longas
SELECT
  pid,
  usename,
  application_name,
  state,
  now() - xact_start AS duration,
  query
FROM pg_stat_activity
WHERE state != 'idle'
  AND xact_start < now() - interval '5 minutes';
```

## Janelas de Manutenção

| Dia | Horário (UTC) | Tarefa |
|-----|---------------|--------|
| Domingo | 03:00 - 05:00 | VACUUM FULL (se necessário) |
| Quarta | 03:00 - 04:00 | REINDEX |
| Diário | 04:00 | Limpeza de dados |

## Procedimento de Emergência

### Banco Lento

1. Verificar conexões bloqueadas:
```sql
SELECT * FROM pg_stat_activity WHERE wait_event IS NOT NULL;
```

2. Terminar queries problemáticas:
```sql
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE pid = <pid>;
```

3. Verificar autovacuum:
```sql
SELECT * FROM pg_stat_progress_vacuum;
```

### Disco Cheio

1. Verificar uso:
```sql
SELECT pg_size_pretty(pg_database_size('techcorp'));
```

2. Limpar dados urgentemente:
```sql
TRUNCATE TABLE auth.sessions; -- se aceitável
```

3. Escalar storage no RDS (requer modificação da instância)

## Links Relacionados

- [Database PostgreSQL](../components/database-postgres.md) - Documentação do banco
- [Resposta a Incidentes](incident-response.md) - Problemas de banco
- [Problemas de Performance](../troubleshooting/performance-issues.md) - Otimização
