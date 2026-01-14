# Database PostgreSQL

## Descrição

O Database PostgreSQL é o sistema de banco de dados relacional principal da TechCorp, utilizado pela maioria dos microsserviços para persistência de dados transacionais. A infraestrutura utiliza Amazon RDS com PostgreSQL 15, configurado para alta disponibilidade com Multi-AZ.

O banco de dados é provisionado como um cluster com uma instância primary para escrita e múltiplas réplicas read-only para distribuição de carga de leitura. Cada microsserviço possui seu próprio schema, garantindo isolamento lógico dos dados.

A estratégia de backup inclui snapshots automáticos diários, point-in-time recovery (PITR) e replicação cross-region para disaster recovery. Todas as conexões são criptografadas e o acesso é restrito por security groups.

## Responsáveis

- **Time:** Data Engineering
- **Tech Lead:** Eduardo Campos
- **Slack:** #data-postgres

## Stack Tecnológica

- Engine: PostgreSQL 15.4
- Hosting: Amazon RDS Multi-AZ
- Connection Pooling: PgBouncer
- Monitoramento: CloudWatch + pg_stat_statements

## Arquitetura

### Instâncias

| Instância | Tipo | Papel | Storage |
|-----------|------|-------|---------|
| `techcorp-primary` | db.r6g.2xlarge | Primary (R/W) | 2TB gp3 |
| `techcorp-replica-1` | db.r6g.xlarge | Read Replica | - |
| `techcorp-replica-2` | db.r6g.xlarge | Read Replica | - |
| `techcorp-analytics` | db.r6g.2xlarge | Analytics Replica | - |

### Schemas por Serviço

| Schema | Serviço Owner | Tabelas Principais |
|--------|---------------|-------------------|
| `auth` | auth-service | users_credentials, sessions, tokens |
| `users` | user-service | users, preferences, addresses |
| `orders` | order-service | orders, order_items, order_history |
| `payments` | payment-service | transactions, refunds |
| `inventory` | inventory-service | stock_levels, reservations |
| `products` | catalog-service | products, categories, attributes |

## Configuração

### Connection Strings

```bash
# Primary (Read/Write)
DATABASE_URL=postgresql://app_user:***@techcorp-primary.xxx.us-east-1.rds.amazonaws.com:5432/techcorp

# Read Replica (Read Only)
DATABASE_URL_RO=postgresql://app_user:***@techcorp-replica-1.xxx.us-east-1.rds.amazonaws.com:5432/techcorp
```

### Parâmetros Importantes

| Parâmetro | Valor | Descrição |
|-----------|-------|-----------|
| `max_connections` | 500 | Conexões máximas por instância |
| `shared_buffers` | 8GB | Memória para cache |
| `work_mem` | 256MB | Memória por operação de sort |
| `maintenance_work_mem` | 1GB | Memória para manutenção |
| `effective_cache_size` | 24GB | Estimativa de cache do SO |

### Configuração do PgBouncer

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

## Conexão Local

### Via psql

```bash
# Conectar ao primary
psql -h techcorp-primary.xxx.rds.amazonaws.com -U app_user -d techcorp

# Conectar via túnel SSH (se fora da VPC)
ssh -L 5432:techcorp-primary.xxx.rds.amazonaws.com:5432 bastion.techcorp.com
psql -h localhost -U app_user -d techcorp
```

### Via GUI

Ferramentas recomendadas:
- DBeaver
- DataGrip
- pgAdmin 4

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/postgres
- **CloudWatch:** https://console.aws.amazon.com/cloudwatch/
- **Performance Insights:** Console RDS

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `CPUUtilization` | Uso de CPU | > 80% |
| `FreeableMemory` | Memória disponível | < 1GB |
| `DatabaseConnections` | Conexões ativas | > 400 |
| `ReadLatency` | Latência de leitura | > 20ms |
| `WriteLatency` | Latência de escrita | > 50ms |
| `ReplicaLag` | Atraso da réplica | > 60s |

### Alertas Configurados

- **PostgresCPUHigh:** CPU acima de 80% por 10 minutos
- **PostgresConnectionsHigh:** Conexões acima de 400
- **PostgresReplicaLagHigh:** Lag acima de 60 segundos
- **PostgresDiskSpaceLow:** Disco abaixo de 20%
- **PostgresDeadlocks:** Deadlocks detectados

## Troubleshooting

### Problema: Queries lentas

**Causa:** Falta de índices, estatísticas desatualizadas ou query mal escrita.

**Solução:**
1. Identificar queries lentas:
   ```sql
   SELECT query, calls, mean_time, total_time
   FROM pg_stat_statements
   ORDER BY mean_time DESC LIMIT 10;
   ```
2. Analisar plano de execução: `EXPLAIN ANALYZE <query>`
3. Verificar índices ausentes
4. Atualizar estatísticas: `ANALYZE <table>`

### Problema: Conexões esgotadas

**Causa:** Leak de conexões, pool mal configurado ou pico de tráfego.

**Solução:**
1. Verificar conexões ativas:
   ```sql
   SELECT count(*), state, application_name
   FROM pg_stat_activity
   GROUP BY state, application_name;
   ```
2. Terminar conexões idle antigas:
   ```sql
   SELECT pg_terminate_backend(pid)
   FROM pg_stat_activity
   WHERE state = 'idle' AND query_start < now() - interval '1 hour';
   ```
3. Verificar configuração do pool nos serviços
4. Aumentar max_connections se necessário

### Problema: Replica lag alto

**Causa:** Primary com carga alta ou rede lenta.

**Solução:**
1. Verificar carga do primary
2. Verificar queries bloqueando replicação
3. Verificar largura de banda de rede
4. Considerar promover réplica se primary com problemas

### Problema: Deadlocks frequentes

**Causa:** Transações concorrentes acessando recursos na ordem errada.

**Solução:**
1. Identificar deadlocks nos logs
2. Analisar queries envolvidas
3. Corrigir ordem de acesso aos recursos
4. Reduzir duração das transações

## Backup e Recovery

### Backup Automático

- Snapshots diários às 03:00 UTC
- Retenção: 7 dias
- PITR disponível para qualquer ponto nos últimos 7 dias

### Recovery Point-in-Time

```bash
# Via AWS CLI
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier techcorp-primary \
  --target-db-instance-identifier techcorp-restored \
  --restore-time "2024-01-15T10:30:00Z"
```

### Recovery de Tabela Específica

Para recovery de dados específicos, usar a analytics replica para evitar impacto em produção:

1. Restaurar backup para instância temporária
2. Exportar dados necessários
3. Importar na instância de produção

## Links Relacionados

- [Manutenção de Banco de Dados](../runbooks/database-maintenance.md) - Procedimentos de manutenção
- [Auth Service](auth-service.md) - Schema auth
- [User Service](user-service.md) - Schema users
- [Order Service](order-service.md) - Schema orders
- [Visão Geral da Arquitetura](../architecture/system-overview.md) - Papel do banco no sistema
