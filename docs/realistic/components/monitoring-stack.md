# Monitoring Stack

## Descrição

O Monitoring Stack é a infraestrutura de observabilidade da TechCorp, composto por Prometheus para coleta de métricas, Grafana para visualização e Alertmanager para gestão de alertas. Este conjunto de ferramentas fornece visibilidade completa sobre a saúde e performance de todos os sistemas.

A stack implementa os três pilares da observabilidade: métricas (Prometheus), logs (ELK Stack) e traces (Jaeger). Isso permite correlacionar informações de diferentes fontes para diagnóstico rápido de problemas.

O sistema de alertas é configurado para notificar os times responsáveis através de múltiplos canais (Slack, PagerDuty, e-mail), com escalação automática para incidentes não reconhecidos.

## Responsáveis

- **Time:** Platform Engineering
- **Tech Lead:** Sandra Oliveira
- **Slack:** #platform-monitoring

## Stack Tecnológica

- Métricas: Prometheus 2.47
- Visualização: Grafana 10.2
- Alertas: Alertmanager 0.26
- Logs: Elasticsearch 8.11 + Kibana 8.11 + Fluentd
- Traces: Jaeger 1.50

## Arquitetura

### Componentes

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
     │   (Métricas)    │ │(Traces│ │    (Logs)     │
     └────────┬────────┘ └───┬───┘ └───────┬───────┘
              │              │              │
     ┌────────▼────────┐     │      ┌──────▼──────┐
     │  Alertmanager   │     │      │   Fluentd   │
     │   (Alertas)     │     │      │ (Coletor)   │
     └────────┬────────┘     │      └─────────────┘
              │              │
     ┌────────▼──────────────▼────────┐
     │     Aplicações / Serviços      │
     └────────────────────────────────┘
```

### Endpoints

| Serviço | URL |
|---------|-----|
| Grafana | https://grafana.techcorp.internal |
| Prometheus | https://prometheus.techcorp.internal |
| Alertmanager | https://alertmanager.techcorp.internal |
| Kibana | https://kibana.techcorp.internal |
| Jaeger | https://jaeger.techcorp.internal |

## Configuração

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

## Dashboards Principais

| Dashboard | Descrição | Link |
|-----------|-----------|------|
| Cluster Overview | Visão geral do K8s | /d/cluster |
| Service Health | Saúde dos microsserviços | /d/services |
| API Gateway | Métricas do gateway | /d/api-gateway |
| PostgreSQL | Métricas do banco | /d/postgres |
| Redis | Métricas do cache | /d/redis |
| RabbitMQ | Métricas das filas | /d/rabbitmq |
| Business Metrics | KPIs de negócio | /d/business |

## Métricas Padrão

### Métricas de Aplicação (RED)

| Métrica | Descrição | Label |
|---------|-----------|-------|
| `http_requests_total` | Total de requisições | method, path, status |
| `http_request_duration_seconds` | Latência | method, path |
| `http_requests_in_progress` | Requisições em andamento | method, path |

### Métricas de Recurso (USE)

| Métrica | Descrição |
|---------|-----------|
| `process_cpu_seconds_total` | Uso de CPU |
| `process_resident_memory_bytes` | Uso de memória |
| `process_open_fds` | File descriptors abertos |

## Alertas Comuns

### Severidade Critical

- Serviço completamente indisponível
- Taxa de erro acima de 50%
- Banco de dados inacessível
- Cluster K8s com problemas

### Severidade Warning

- Latência P99 acima do SLO
- Taxa de erro acima de 5%
- Uso de recursos acima de 80%
- Replica lag alto

### Severidade Info

- Deploy realizado
- Scaling automático ativado
- Backup completado

## Troubleshooting

### Problema: Métricas não aparecendo

**Causa:** Target não sendo scraped ou aplicação não expondo métricas.

**Solução:**
1. Verificar targets no Prometheus: Status > Targets
2. Verificar se pod tem annotation correta:
   ```yaml
   annotations:
     prometheus.io/scrape: "true"
     prometheus.io/port: "8080"
   ```
3. Verificar se endpoint /metrics está acessível
4. Verificar logs do Prometheus

### Problema: Dashboard lento

**Causa:** Query muito pesada ou período muito longo.

**Solução:**
1. Reduzir período de visualização
2. Otimizar queries (usar recording rules)
3. Aumentar step interval
4. Verificar cardinalidade das métricas

### Problema: Alertas não chegando

**Causa:** Configuração incorreta do Alertmanager ou canal indisponível.

**Solução:**
1. Verificar se alerta está firing: Alertmanager > Alerts
2. Verificar rotas de notificação
3. Testar receiver manualmente
4. Verificar logs do Alertmanager

### Problema: Logs não aparecendo no Kibana

**Causa:** Fluentd não coletando ou Elasticsearch com problemas.

**Solução:**
1. Verificar pods do Fluentd: `kubectl get pods -n logging`
2. Verificar índices no Elasticsearch
3. Verificar filtros de parsing do Fluentd
4. Verificar espaço em disco do Elasticsearch

## Boas Práticas

### Instrumentação

- Use labels consistentes em todas as métricas
- Evite labels com alta cardinalidade (IDs, timestamps)
- Implemente os padrões RED e USE
- Adicione métricas de negócio significativas

### Alertas

- Alerte sobre sintomas, não causas
- Use severidades apropriadas
- Configure runbooks para cada alerta
- Evite alert fatigue (muitos alertas)

### Dashboards

- Organize dashboards por serviço/time
- Use variáveis para filtros comuns
- Inclua links para logs e traces
- Documente métricas não óbvias

## Links Relacionados

- [Kubernetes Cluster](kubernetes-cluster.md) - Métricas do cluster
- [Database PostgreSQL](database-postgres.md) - Métricas do banco
- [Cache Service](cache-service.md) - Métricas do Redis
- [Queue Service](queue-service.md) - Métricas do RabbitMQ
- [Resposta a Incidentes](../runbooks/incident-response.md) - Uso em incidentes
