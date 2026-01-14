# Database PostgreSQL

## Description

The Database PostgreSQL is TechCorp's primary relational database system, used by most microservices for transactional data persistence. The infrastructure uses Amazon RDS with PostgreSQL 15, configured for high availability with Multi-AZ.

The database is provisioned as a cluster with a primary instance for writes and multiple read-only replicas for distributing read load. Each microservice has its own schema, ensuring logical data isolation.

The backup strategy includes automatic daily snapshots, point-in-time recovery (PITR), and cross-region replication for disaster recovery. All connections are encrypted and access is restricted by security groups.

## Owners

- **Team:** Data Engineering
- **Tech Lead:** Eduardo Campos
- **Slack:** #data-postgres

## Technology Stack

- Engine: PostgreSQL 15.4
- Hosting: Amazon RDS Multi-AZ
- Connection Pooling: PgBouncer
- Monitoring: CloudWatch + pg_stat_statements

## Architecture

### Instances

| Instance | Type | Role | Storage |
|----------|------|------|---------|
| `techcorp-primary` | db.r6g.2xlarge | Primary (R/W) | 2TB gp3 |
| `techcorp-replica-1` | db.r6g.xlarge | Read Replica | - |
| `techcorp-replica-2` | db.r6g.xlarge | Read Replica | - |
| `techcorp-analytics` | db.r6g.2xlarge | Analytics Replica | - |

### Schemas by Service

| Schema | Owner Service | Main Tables |
|--------|---------------|-------------|
| `auth` | auth-service | users_credentials, sessions, tokens |
| `users` | user-service | users, preferences, addresses |
| `orders` | order-service | orders, order_items, order_history |
| `payments` | payment-service | transactions, refunds |
| `inventory` | inventory-service | stock_levels, reservations |
| `products` | catalog-service | products, categories, attributes |

## Configuration

### Connection Strings

```bash
# Primary (Read/Write)
DATABASE_URL=postgresql://app_user:***@techcorp-primary.xxx.us-east-1.rds.amazonaws.com:5432/techcorp

# Read Replica (Read Only)
DATABASE_URL_RO=postgresql://app_user:***@techcorp-replica-1.xxx.us-east-1.rds.amazonaws.com:5432/techcorp
```

### Important Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `max_connections` | 500 | Maximum connections per instance |
| `shared_buffers` | 8GB | Memory for cache |
| `work_mem` | 256MB | Memory per sort operation |
| `maintenance_work_mem` | 1GB | Memory for maintenance |
| `effective_cache_size` | 24GB | OS cache estimate |

### PgBouncer Configuration

```ini
# pgbouncer.ini
[databases]
techcorp = host=techcorp-primary.xxx.rds.amazonaws.com dbname=techcorp

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 50
reserve_pool_size = 10
```

## Local Connection

### Via psql

```bash
# Connect to primary
psql -h techcorp-primary.xxx.rds.amazonaws.com -U app_user -d techcorp

# Connect via SSH tunnel (if outside VPC)
ssh -L 5432:techcorp-primary.xxx.rds.amazonaws.com:5432 bastion.techcorp.com
psql -h localhost -U app_user -d techcorp
```

### Via GUI

Recommended tools:
- DBeaver
- DataGrip
- pgAdmin 4

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/postgres
- **CloudWatch:** https://console.aws.amazon.com/cloudwatch/
- **Performance Insights:** RDS Console

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `CPUUtilization` | CPU usage | > 80% |
| `FreeableMemory` | Available memory | < 1GB |
| `DatabaseConnections` | Active connections | > 400 |
| `ReadLatency` | Read latency | > 20ms |
| `WriteLatency` | Write latency | > 50ms |
| `ReplicaLag` | Replica delay | > 60s |

### Configured Alerts

- **PostgresCPUHigh:** CPU above 80% for 10 minutes
- **PostgresConnectionsHigh:** Connections above 400
- **PostgresReplicaLagHigh:** Lag above 60 seconds
- **PostgresDiskSpaceLow:** Disk below 20%
- **PostgresDeadlocks:** Deadlocks detected

## Troubleshooting

### Issue: Slow queries

**Cause:** Missing indexes, outdated statistics, or poorly written query.

**Solution:**
1. Identify slow queries:
   ```sql
   SELECT query, calls, mean_time, total_time
   FROM pg_stat_statements
   ORDER BY mean_time DESC LIMIT 10;
   ```
2. Analyze execution plan: `EXPLAIN ANALYZE <query>`
3. Check for missing indexes
4. Update statistics: `ANALYZE <table>`

### Issue: Connections exhausted

**Cause:** Connection leak, misconfigured pool, or traffic spike.

**Solution:**
1. Check active connections:
   ```sql
   SELECT count(*), state, application_name
   FROM pg_stat_activity
   GROUP BY state, application_name;
   ```
2. Terminate old idle connections:
   ```sql
   SELECT pg_terminate_backend(pid)
   FROM pg_stat_activity
   WHERE state = 'idle' AND query_start < now() - interval '1 hour';
   ```
3. Check pool configuration in services
4. Increase max_connections if necessary

### Issue: High replica lag

**Cause:** Primary under high load or slow network.

**Solution:**
1. Check primary load
2. Check queries blocking replication
3. Check network bandwidth
4. Consider promoting replica if primary has issues

### Issue: Frequent deadlocks

**Cause:** Concurrent transactions accessing resources in wrong order.

**Solution:**
1. Identify deadlocks in logs
2. Analyze involved queries
3. Fix resource access order
4. Reduce transaction duration

## Backup and Recovery

### Automatic Backup

- Daily snapshots at 03:00 UTC
- Retention: 7 days
- PITR available for any point in the last 7 days

### Point-in-Time Recovery

```bash
# Via AWS CLI
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier techcorp-primary \
  --target-db-instance-identifier techcorp-restored \
  --restore-time "2024-01-15T10:30:00Z"
```

### Specific Table Recovery

For recovering specific data, use the analytics replica to avoid production impact:

1. Restore backup to temporary instance
2. Export required data
3. Import into production instance

## Related Links

- [Database Maintenance](../runbooks/database-maintenance.md) - Maintenance procedures
- [Auth Service](auth-service.md) - Auth schema
- [User Service](user-service.md) - Users schema
- [Order Service](order-service.md) - Orders schema
- [System Overview](../architecture/system-overview.md) - Database role in the system
