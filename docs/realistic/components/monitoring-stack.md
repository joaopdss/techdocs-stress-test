# Monitoring Stack

## Description

The Monitoring Stack is TechCorp's observability infrastructure, composed of Prometheus for metrics collection, Grafana for visualization, and Alertmanager for alert management. This set of tools provides complete visibility into the health and performance of all systems.

The stack implements the three pillars of observability: metrics (Prometheus), logs (ELK Stack), and traces (Jaeger). This allows correlating information from different sources for rapid problem diagnosis.

The alerting system is configured to notify responsible teams through multiple channels (Slack, PagerDuty, email), with automatic escalation for unacknowledged incidents.

## Owners

- **Team:** Platform Engineering
- **Tech Lead:** Sandra Oliveira
- **Slack:** #platform-monitoring

## Technology Stack

- Metrics: Prometheus 2.47
- Visualization: Grafana 10.2
- Alerts: Alertmanager 0.26
- Logs: Elasticsearch 8.11 + Kibana 8.11 + Fluentd
- Traces: Jaeger 1.50

## Architecture

### Components

```
                    ┌─────────────────┐
                    │    Grafana      │
                    │  (Dashboards)   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼────────┐ ┌───▼───┐ ┌───────▼───────┐
     │   Prometheus    │ │Jaeger │ │ Elasticsearch │
     │   (Metrics)     │ │(Traces│ │    (Logs)     │
     └────────┬────────┘ └───┬───┘ └───────┬───────┘
              │              │              │
     ┌────────▼────────┐     │      ┌──────▼──────┐
     │  Alertmanager   │     │      │   Fluentd   │
     │   (Alerts)      │     │      │ (Collector) │
     └────────┬────────┘     │      └─────────────┘
              │              │
     ┌────────▼──────────────▼────────┐
     │     Applications / Services    │
     └────────────────────────────────┘
```

### Endpoints

| Service | URL |
|---------|-----|
| Grafana | https://grafana.techcorp.internal |
| Prometheus | https://prometheus.techcorp.internal |
| Alertmanager | https://alertmanager.techcorp.internal |
| Kibana | https://kibana.techcorp.internal |
| Jaeger | https://jaeger.techcorp.internal |

## Configuration

### Prometheus

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### Alertmanager

```yaml
# alertmanager.yml
route:
  receiver: 'slack-default'
  group_by: ['alertname', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
    - match:
        team: platform
      receiver: 'slack-platform'

receivers:
  - name: 'slack-default'
    slack_configs:
      - channel: '#alerts-general'
  - name: 'slack-platform'
    slack_configs:
      - channel: '#platform-alerts'
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: <key>
```

## Main Dashboards

| Dashboard | Description | Link |
|-----------|-------------|------|
| Cluster Overview | K8s overview | /d/cluster |
| Service Health | Microservices health | /d/services |
| API Gateway | Gateway metrics | /d/api-gateway |
| PostgreSQL | Database metrics | /d/postgres |
| Redis | Cache metrics | /d/redis |
| RabbitMQ | Queue metrics | /d/rabbitmq |
| Business Metrics | Business KPIs | /d/business |

## Standard Metrics

### Application Metrics (RED)

| Metric | Description | Label |
|--------|-------------|-------|
| `http_requests_total` | Total requests | method, path, status |
| `http_request_duration_seconds` | Latency | method, path |
| `http_requests_in_progress` | In-progress requests | method, path |

### Resource Metrics (USE)

| Metric | Description |
|--------|-------------|
| `process_cpu_seconds_total` | CPU usage |
| `process_resident_memory_bytes` | Memory usage |
| `process_open_fds` | Open file descriptors |

## Common Alerts

### Critical Severity

- Service completely unavailable
- Error rate above 50%
- Database inaccessible
- K8s cluster with issues

### Warning Severity

- P99 latency above SLO
- Error rate above 5%
- Resource usage above 80%
- High replica lag

### Info Severity

- Deploy completed
- Auto scaling activated
- Backup completed

## Troubleshooting

### Issue: Metrics not appearing

**Cause:** Target not being scraped or application not exposing metrics.

**Solution:**
1. Check targets in Prometheus: Status > Targets
2. Verify pod has correct annotation:
   ```yaml
   annotations:
     prometheus.io/scrape: "true"
     prometheus.io/port: "8080"
   ```
3. Verify /metrics endpoint is accessible
4. Check Prometheus logs

### Issue: Slow dashboard

**Cause:** Heavy query or very long period.

**Solution:**
1. Reduce visualization period
2. Optimize queries (use recording rules)
3. Increase step interval
4. Check metrics cardinality

### Issue: Alerts not arriving

**Cause:** Incorrect Alertmanager configuration or unavailable channel.

**Solution:**
1. Check if alert is firing: Alertmanager > Alerts
2. Verify notification routes
3. Test receiver manually
4. Check Alertmanager logs

### Issue: Logs not appearing in Kibana

**Cause:** Fluentd not collecting or Elasticsearch issues.

**Solution:**
1. Check Fluentd pods: `kubectl get pods -n logging`
2. Check indexes in Elasticsearch
3. Verify Fluentd parsing filters
4. Check Elasticsearch disk space

## Best Practices

### Instrumentation

- Use consistent labels across all metrics
- Avoid labels with high cardinality (IDs, timestamps)
- Implement RED and USE patterns
- Add meaningful business metrics

### Alerts

- Alert on symptoms, not causes
- Use appropriate severities
- Configure runbooks for each alert
- Avoid alert fatigue (too many alerts)

### Dashboards

- Organize dashboards by service/team
- Use variables for common filters
- Include links to logs and traces
- Document non-obvious metrics

## Related Links

- [Kubernetes Cluster](kubernetes-cluster.md) - Cluster metrics
- [Database PostgreSQL](database-postgres.md) - Database metrics
- [Cache Service](cache-service.md) - Redis metrics
- [Queue Service](queue-service.md) - RabbitMQ metrics
- [Incident Response](../runbooks/incident-response.md) - Usage in incidents
