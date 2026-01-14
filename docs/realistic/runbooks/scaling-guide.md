# Guia de Escalar Serviços

## Visão Geral

Este guia descreve os procedimentos para escalar serviços na plataforma TechCorp. Escalar recursos é necessário quando a demanda excede a capacidade atual, seja de forma planejada (eventos, campanhas) ou reativa (picos inesperados).

## Tipos de Escalabilidade

### Escalabilidade Horizontal

Aumentar o número de instâncias (pods) de um serviço.

- **Vantagens:** Não requer downtime, distribuição de carga
- **Quando usar:** Aumento de tráfego, processamento paralelo

### Escalabilidade Vertical

Aumentar recursos (CPU, memória) de cada instância.

- **Vantagens:** Simples de implementar
- **Quando usar:** Operações memory-intensive, limitações de concorrência

## Escalar Manualmente

### Escalar Pods (Horizontal)

```bash
# Verificar estado atual
kubectl get deployment <app-name> -n production

# Escalar para número específico de réplicas
kubectl scale deployment <app-name> -n production --replicas=10

# Verificar progresso
kubectl rollout status deployment/<app-name> -n production
```

### Escalar Recursos (Vertical)

```bash
# Editar deployment
kubectl edit deployment <app-name> -n production

# Ou aplicar patch
kubectl patch deployment <app-name> -n production --patch '
spec:
  template:
    spec:
      containers:
      - name: <app-name>
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
'
```

## Horizontal Pod Autoscaler (HPA)

O HPA ajusta automaticamente o número de pods baseado em métricas.

### Verificar HPA Existente

```bash
# Listar HPAs
kubectl get hpa -n production

# Detalhes de um HPA específico
kubectl describe hpa <app-name> -n production
```

### Criar/Atualizar HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
```

```bash
# Aplicar configuração
kubectl apply -f hpa.yaml
```

### Ajustar Limites do HPA

```bash
# Aumentar máximo de réplicas temporariamente
kubectl patch hpa <app-name> -n production --patch '{"spec":{"maxReplicas": 30}}'

# Aumentar mínimo de réplicas para evento
kubectl patch hpa <app-name> -n production --patch '{"spec":{"minReplicas": 10}}'
```

## Escalar por Serviço

### API Gateway

```bash
# Gateway é crítico - escalar agressivamente
kubectl scale deployment api-gateway -n production --replicas=15

# Verificar conexões
kubectl top pods -n production -l app=api-gateway
```

### Auth Service

```bash
# Auth tem estado em Redis - escalar é seguro
kubectl scale deployment auth-service -n production --replicas=8
```

### Order Service

```bash
# Order usa sagas - escalar com cuidado
# Verificar jobs pendentes antes
kubectl logs -n production -l app=order-service | grep "saga"

# Escalar
kubectl scale deployment order-service -n production --replicas=10
```

### Payment Service

```bash
# Payment tem rate limits de providers
# Verificar limites antes de escalar
kubectl scale deployment payment-service -n production --replicas=6
```

### Workers de Fila

```bash
# Workers podem escalar livremente
kubectl scale deployment notification-worker -n production --replicas=20
```

## Escalar Infraestrutura

### Escalar Cluster Kubernetes (Nodes)

```bash
# Via AWS Console ou CLI
aws eks update-nodegroup-config \
  --cluster-name techcorp-production \
  --nodegroup-name general \
  --scaling-config minSize=5,maxSize=20,desiredSize=10

# Verificar nodes
kubectl get nodes
```

### Escalar Redis

```bash
# Redis Cluster - adicionar shards
# Via AWS Console: ElastiCache > Clusters > Modify

# Verificar status
aws elasticache describe-cache-clusters --cache-cluster-id techcorp-redis
```

### Escalar PostgreSQL

```bash
# RDS Read Replicas - adicionar réplica
aws rds create-db-instance-read-replica \
  --db-instance-identifier techcorp-replica-3 \
  --source-db-instance-identifier techcorp-primary

# Escalar verticalmente (requer restart)
aws rds modify-db-instance \
  --db-instance-identifier techcorp-primary \
  --db-instance-class db.r6g.4xlarge \
  --apply-immediately
```

### Escalar RabbitMQ

```bash
# Verificar uso atual
kubectl exec -it rabbitmq-0 -n production -- rabbitmqctl status

# Escalar cluster
kubectl scale statefulset rabbitmq -n production --replicas=5
```

## Planejamento para Eventos

### Checklist Pré-Evento

- [ ] Identificar serviços críticos para o evento
- [ ] Calcular tráfego esperado (baseline * multiplicador)
- [ ] Ajustar minReplicas do HPA
- [ ] Ajustar maxReplicas do HPA
- [ ] Escalar nodes do cluster
- [ ] Escalar banco de dados (se necessário)
- [ ] Escalar Redis (se necessário)
- [ ] Comunicar time de plantão

### Exemplo: Black Friday

```bash
# 1. Escalar nodes (fazer 1 dia antes)
aws eks update-nodegroup-config \
  --cluster-name techcorp-production \
  --nodegroup-name general \
  --scaling-config minSize=10,maxSize=50,desiredSize=25

# 2. Ajustar HPAs
for app in api-gateway auth-service user-service order-service payment-service; do
  kubectl patch hpa $app -n production --patch '{"spec":{"minReplicas": 10, "maxReplicas": 50}}'
done

# 3. Escalar workers
kubectl scale deployment notification-worker -n production --replicas=30
kubectl scale deployment inventory-worker -n production --replicas=20
```

### Pós-Evento

```bash
# Retornar configurações ao normal
for app in api-gateway auth-service user-service order-service payment-service; do
  kubectl patch hpa $app -n production --patch '{"spec":{"minReplicas": 3, "maxReplicas": 20}}'
done

# O HPA reduzirá pods automaticamente após estabilização
```

## Monitoramento Durante Scaling

### Métricas Importantes

```bash
# Uso de recursos
kubectl top pods -n production

# Status do HPA
kubectl get hpa -n production -w

# Eventos do cluster
kubectl get events -n production --sort-by='.lastTimestamp'
```

### Dashboard Grafana

- **Cluster Overview:** Visão geral de recursos
- **Pod Resources:** CPU/Memory por pod
- **HPA Status:** Réplicas desejadas vs atuais

## Troubleshooting

### Pods não escalam

**Causa:** Recursos insuficientes no cluster.

**Solução:**
1. Verificar eventos: `kubectl get events -n production`
2. Verificar capacity: `kubectl describe nodes`
3. Escalar node group se necessário

### HPA não está funcionando

**Causa:** Metrics server com problema ou métricas não disponíveis.

**Solução:**
1. Verificar metrics-server: `kubectl get pods -n kube-system | grep metrics`
2. Verificar métricas: `kubectl top pods -n production`
3. Verificar events do HPA: `kubectl describe hpa <name>`

### Escalar causa instabilidade

**Causa:** Dependência não escalou junto ou rate limit atingido.

**Solução:**
1. Escalar dependências proporcionalmente
2. Verificar connection pools
3. Verificar rate limits de APIs externas

## Links Relacionados

- [Guia de Deploy](deploy-guide.md) - Deploy de serviços
- [Resposta a Incidentes](incident-response.md) - Uso emergencial
- [Kubernetes Cluster](../components/kubernetes-cluster.md) - Infraestrutura
- [Monitoring Stack](../components/monitoring-stack.md) - Monitoramento
