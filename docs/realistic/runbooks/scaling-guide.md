# Scaling Guide

## Overview

This guide describes the procedures for scaling services on the TechCorp platform. Scaling resources is necessary when demand exceeds current capacity, whether planned (events, campaigns) or reactive (unexpected spikes).

## Types of Scaling

### Horizontal Scaling

Increasing the number of instances (pods) of a service.

- **Advantages:** No downtime required, load distribution
- **When to use:** Traffic increase, parallel processing

### Vertical Scaling

Increasing resources (CPU, memory) of each instance.

- **Advantages:** Simple to implement
- **When to use:** Memory-intensive operations, concurrency limitations

## Manual Scaling

### Scale Pods (Horizontal)

```bash
# Check current state
kubectl get deployment <app-name> -n production

# Scale to specific number of replicas
kubectl scale deployment <app-name> -n production --replicas=10

# Check progress
kubectl rollout status deployment/<app-name> -n production
```

### Scale Resources (Vertical)

```bash
# Edit deployment
kubectl edit deployment <app-name> -n production

# Or apply patch
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

The HPA automatically adjusts the number of pods based on metrics.

### Check Existing HPA

```bash
# List HPAs
kubectl get hpa -n production

# Details of a specific HPA
kubectl describe hpa <app-name> -n production
```

### Create/Update HPA

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
# Apply configuration
kubectl apply -f hpa.yaml
```

### Adjust HPA Limits

```bash
# Temporarily increase maximum replicas
kubectl patch hpa <app-name> -n production --patch '{"spec":{"maxReplicas": 30}}'

# Increase minimum replicas for event
kubectl patch hpa <app-name> -n production --patch '{"spec":{"minReplicas": 10}}'
```

## Scaling by Service

### API Gateway

```bash
# Gateway is critical - scale aggressively
kubectl scale deployment api-gateway -n production --replicas=15

# Check connections
kubectl top pods -n production -l app=api-gateway
```

### Auth Service

```bash
# Auth has state in Redis - scaling is safe
kubectl scale deployment auth-service -n production --replicas=8
```

### Order Service

```bash
# Order uses sagas - scale carefully
# Check pending jobs first
kubectl logs -n production -l app=order-service | grep "saga"

# Scale
kubectl scale deployment order-service -n production --replicas=10
```

### Payment Service

```bash
# Payment has provider rate limits
# Check limits before scaling
kubectl scale deployment payment-service -n production --replicas=6
```

### Queue Workers

```bash
# Workers can scale freely
kubectl scale deployment notification-worker -n production --replicas=20
```

## Infrastructure Scaling

### Scale Kubernetes Cluster (Nodes)

```bash
# Via AWS Console or CLI
aws eks update-nodegroup-config \
  --cluster-name techcorp-production \
  --nodegroup-name general \
  --scaling-config minSize=5,maxSize=20,desiredSize=10

# Check nodes
kubectl get nodes
```

### Scale Redis

```bash
# Redis Cluster - add shards
# Via AWS Console: ElastiCache > Clusters > Modify

# Check status
aws elasticache describe-cache-clusters --cache-cluster-id techcorp-redis
```

### Scale PostgreSQL

```bash
# RDS Read Replicas - add replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier techcorp-replica-3 \
  --source-db-instance-identifier techcorp-primary

# Scale vertically (requires restart)
aws rds modify-db-instance \
  --db-instance-identifier techcorp-primary \
  --db-instance-class db.r6g.4xlarge \
  --apply-immediately
```

### Scale RabbitMQ

```bash
# Check current usage
kubectl exec -it rabbitmq-0 -n production -- rabbitmqctl status

# Scale cluster
kubectl scale statefulset rabbitmq -n production --replicas=5
```

## Planning for Events

### Pre-Event Checklist

- [ ] Identify critical services for the event
- [ ] Calculate expected traffic (baseline * multiplier)
- [ ] Adjust HPA minReplicas
- [ ] Adjust HPA maxReplicas
- [ ] Scale cluster nodes
- [ ] Scale database (if necessary)
- [ ] Scale Redis (if necessary)
- [ ] Communicate with on-call team

### Example: Black Friday

```bash
# 1. Scale nodes (do 1 day before)
aws eks update-nodegroup-config \
  --cluster-name techcorp-production \
  --nodegroup-name general \
  --scaling-config minSize=10,maxSize=50,desiredSize=25

# 2. Adjust HPAs
for app in api-gateway auth-service user-service order-service payment-service; do
  kubectl patch hpa $app -n production --patch '{"spec":{"minReplicas": 10, "maxReplicas": 50}}'
done

# 3. Scale workers
kubectl scale deployment notification-worker -n production --replicas=30
kubectl scale deployment inventory-worker -n production --replicas=20
```

### Post-Event

```bash
# Return configurations to normal
for app in api-gateway auth-service user-service order-service payment-service; do
  kubectl patch hpa $app -n production --patch '{"spec":{"minReplicas": 3, "maxReplicas": 20}}'
done

# The HPA will automatically reduce pods after stabilization
```

## Monitoring During Scaling

### Important Metrics

```bash
# Resource usage
kubectl top pods -n production

# HPA status
kubectl get hpa -n production -w

# Cluster events
kubectl get events -n production --sort-by='.lastTimestamp'
```

### Grafana Dashboard

- **Cluster Overview:** Resource overview
- **Pod Resources:** CPU/Memory per pod
- **HPA Status:** Desired vs current replicas

## Troubleshooting

### Pods not scaling

**Cause:** Insufficient resources in cluster.

**Solution:**
1. Check events: `kubectl get events -n production`
2. Check capacity: `kubectl describe nodes`
3. Scale node group if necessary

### HPA not working

**Cause:** Metrics server issue or metrics unavailable.

**Solution:**
1. Check metrics-server: `kubectl get pods -n kube-system | grep metrics`
2. Check metrics: `kubectl top pods -n production`
3. Check HPA events: `kubectl describe hpa <name>`

### Scaling causes instability

**Cause:** Dependency didn't scale together or rate limit reached.

**Solution:**
1. Scale dependencies proportionally
2. Check connection pools
3. Check rate limits of external APIs

## Related Links

- [Deploy Guide](deploy-guide.md) - Deploying services
- [Incident Response](incident-response.md) - Emergency usage
- [Kubernetes Cluster](../components/kubernetes-cluster.md) - Infrastructure
- [Monitoring Stack](../components/monitoring-stack.md) - Monitoring
