# Database Maintenance

## Overview

This guide describes PostgreSQL maintenance procedures at TechCorp. Regular maintenance is essential to ensure performance, data integrity, and system availability.

## Maintenance Tasks

### Automatic Maintenance

| Task | Frequency | Time | Impact |
|------|-----------|------|--------|
| Auto VACUUM | Continuous | - | Low |
| Incremental backup | Hourly | - | None |
| Full backup | Daily | 03:00 UTC | Low |
| Statistics update | Continuous | - | None |

### Manual Maintenance

| Task | Frequency | Window | Impact |
|------|-----------|--------|--------|
| VACUUM FULL | Quarterly | Night | High |
| REINDEX | Monthly | Night | Medium |
| Slow query analysis | Weekly | - | None |
| Old data cleanup | Monthly | Night | Low |

## Procedures

### Check Database Health

```sql
-- Check table bloat
SELECT
  schemaname || '.' || tablename AS table,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
  pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_size,
  pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) AS index_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;

-- Check unused indexes
SELECT
  schemaname || '.' || relname AS table,
  indexrelname AS index,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size,
  idx_scan AS scans
FROM pg_stat_user_indexes
WHERE idx_scan < 50
ORDER BY pg_relation_size(indexrelid) DESC;

-- Check active locks
SELECT
  pid,
  usename,
  pg_blocking_pids(pid) AS blocked_by,
  query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### VACUUM

#### Regular VACUUM

Executed automatically by autovacuum. To force:

```sql
-- VACUUM on specific table
VACUUM VERBOSE orders.orders;

-- VACUUM on entire schema
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

**WARNING:** VACUUM FULL locks the table. Schedule during maintenance window.

```sql
-- Check if needed
SELECT
  schemaname || '.' || relname AS table,
  n_dead_tup AS dead_tuples,
  pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_stat_user_tables
WHERE n_dead_tup > 100000
ORDER BY n_dead_tup DESC;

-- Execute VACUUM FULL
VACUUM FULL VERBOSE orders.orders;
```

### REINDEX

```sql
-- REINDEX specific index
REINDEX INDEX CONCURRENTLY orders.orders_user_id_idx;

-- REINDEX table (all indexes)
REINDEX TABLE CONCURRENTLY orders.orders;

-- REINDEX entire schema
REINDEX SCHEMA CONCURRENTLY orders;
```

**NOTE:** Use `CONCURRENTLY` to avoid locking in production.

### Slow Query Analysis

#### Enable pg_stat_statements

```sql
-- Check if extension is active
SELECT * FROM pg_extension WHERE extname = 'pg_stat_statements';

-- Top 10 queries by total time
SELECT
  substring(query, 1, 100) AS query,
  calls,
  round(total_exec_time::numeric, 2) AS total_time_ms,
  round(mean_exec_time::numeric, 2) AS mean_time_ms,
  rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Queries with high variance (unstable)
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

#### Specific Query Analysis

```sql
-- Execution plan
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders.orders WHERE user_id = 'uuid' ORDER BY created_at DESC LIMIT 10;
```

### Old Data Cleanup

#### Retention Configuration

| Schema | Table | Retention | Criteria |
|--------|-------|-----------|----------|
| auth | sessions | 30 days | expired_at |
| orders | order_events | 1 year | created_at |
| inventory | reservations | 7 days | expires_at |
| audit | logs | 2 years | created_at |

#### Cleanup Script

```sql
-- Clean expired sessions
DELETE FROM auth.sessions
WHERE expired_at < NOW() - INTERVAL '30 days';

-- Clean expired reservations
DELETE FROM inventory.reservations
WHERE status = 'EXPIRED' AND expires_at < NOW() - INTERVAL '7 days';

-- Clean old order events (in batches)
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

    -- Pause between batches to avoid overload
    PERFORM pg_sleep(1);
  END LOOP;
END;
$$;
```

### Backup and Recovery

#### Verify Backups

```bash
# List available snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier techcorp-primary \
  --query 'DBSnapshots[*].[DBSnapshotIdentifier,SnapshotCreateTime,Status]' \
  --output table

# Check PITR
aws rds describe-db-instances \
  --db-instance-identifier techcorp-primary \
  --query 'DBInstances[*].LatestRestorableTime'
```

#### Restore from Backup

```bash
# Point-in-time restore (creates new instance)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier techcorp-primary \
  --target-db-instance-identifier techcorp-restored \
  --restore-time "2024-01-15T10:30:00Z"

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier techcorp-restored \
  --db-snapshot-identifier techcorp-daily-20240115
```

#### Export Specific Table

```bash
# Via pg_dump
pg_dump -h techcorp-replica.xxx.rds.amazonaws.com \
  -U app_user -d techcorp \
  -t orders.orders \
  -f orders_backup.sql

# Import
psql -h techcorp-primary.xxx.rds.amazonaws.com \
  -U app_user -d techcorp \
  -f orders_backup.sql
```

### Monitoring

#### Maintenance Alerts

| Condition | Threshold | Action |
|-----------|-----------|--------|
| Dead tuples > 10M | Warning | Schedule VACUUM |
| Bloat > 30% | Warning | Schedule VACUUM FULL |
| Connections > 80% | Critical | Investigate |
| Replica lag > 60s | Critical | Investigate |

#### Dashboard Queries

```sql
-- Active connections by application
SELECT
  application_name,
  count(*) AS connections,
  count(*) FILTER (WHERE state = 'active') AS active,
  count(*) FILTER (WHERE state = 'idle') AS idle
FROM pg_stat_activity
WHERE datname = 'techcorp'
GROUP BY application_name;

-- Long transactions
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

## Maintenance Windows

| Day | Time (UTC) | Task |
|-----|------------|------|
| Sunday | 03:00 - 05:00 | VACUUM FULL (if necessary) |
| Wednesday | 03:00 - 04:00 | REINDEX |
| Daily | 04:00 | Data cleanup |

## Emergency Procedures

### Slow Database

1. Check blocked connections:
```sql
SELECT * FROM pg_stat_activity WHERE wait_event IS NOT NULL;
```

2. Terminate problematic queries:
```sql
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE pid = <pid>;
```

3. Check autovacuum:
```sql
SELECT * FROM pg_stat_progress_vacuum;
```

### Disk Full

1. Check usage:
```sql
SELECT pg_size_pretty(pg_database_size('techcorp'));
```

2. Urgently clean data:
```sql
TRUNCATE TABLE auth.sessions; -- if acceptable
```

3. Scale storage in RDS (requires instance modification)

## Related Links

- [Database PostgreSQL](../components/database-postgres.md) - Database documentation
- [Incident Response](incident-response.md) - Database issues
- [Performance Issues](../troubleshooting/performance-issues.md) - Optimization
