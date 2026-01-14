# Kubernetes Cluster

## Description

The Kubernetes Cluster is TechCorp's container orchestration platform, responsible for running all microservices in the production environment. The cluster manages the application lifecycle, including deployment, scaling, networking, and self-healing.

The infrastructure uses managed clusters (EKS on AWS), allowing the platform team to focus on application operations rather than control plane maintenance. The cluster is configured with multiple node groups optimized for different workload types.

The architecture implements namespace isolation, with separate environments for production, staging, and development. Network Policies ensure that services only communicate as permitted.

## Owners

- **Team:** Platform Engineering
- **Tech Lead:** Roberto Nascimento
- **Slack:** #platform-k8s

## Technology Stack

- Orchestration: Kubernetes 1.28 (EKS)
- Service Mesh: Istio 1.19
- Ingress: AWS ALB Ingress Controller
- Secrets: External Secrets Operator + AWS Secrets Manager
- GitOps: ArgoCD
- Monitoring: Prometheus + Grafana

## Cluster Architecture

### Node Groups

| Node Group | Instance Type | Purpose | Scaling |
|------------|---------------|---------|---------|
| `general` | m6i.xlarge | General workloads | 3-10 |
| `compute` | c6i.2xlarge | CPU-intensive | 2-8 |
| `memory` | r6i.xlarge | Memory-intensive | 2-6 |
| `spot` | m6i.xlarge (spot) | Batch jobs | 0-20 |

### Namespaces

| Namespace | Description |
|-----------|-------------|
| `production` | Production services |
| `staging` | Staging environment |
| `monitoring` | Prometheus, Grafana |
| `istio-system` | Service mesh |
| `argocd` | GitOps controller |
| `external-secrets` | Secrets operator |

## Configuration

### Cluster Access

```bash
# Configure kubeconfig
aws eks update-kubeconfig --name techcorp-production --region us-east-1

# Verify connection
kubectl cluster-info

# List namespaces
kubectl get namespaces
```

### Available Contexts

```bash
# Production
kubectl config use-context techcorp-production

# Staging
kubectl config use-context techcorp-staging
```

## Deployment Patterns

### Manifest Structure

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        version: v1.2.3
    spec:
      containers:
        - name: user-service
          image: techcorp/user-service:v1.2.3
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
```

### Resource Quotas per Namespace

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "50"
    requests.memory: "100Gi"
    limits.cpu: "100"
    limits.memory: "200Gi"
    pods: "200"
```

## Monitoring

- **Grafana Dashboard:** https://grafana.techcorp.internal/d/kubernetes
- **ArgoCD:** https://argocd.techcorp.internal
- **Prometheus Metrics:** https://prometheus.techcorp.internal

### Key Metrics

| Metric | Description | Alert |
|--------|-------------|-------|
| `kube_pod_status_ready` | Pods in Ready state | < expected |
| `kube_node_status_condition` | Node health | NotReady |
| `container_memory_usage_bytes` | Memory usage | > 80% limit |
| `container_cpu_usage_seconds` | CPU usage | > 80% limit |

### Configured Alerts

- **KubePodNotReady:** Pod not Ready for more than 5 minutes
- **KubeNodeNotReady:** Node not Ready for more than 5 minutes
- **KubePodCrashLooping:** Pod restarting repeatedly
- **KubeMemoryOvercommit:** Cluster with memory overcommit
- **KubeCPUOvercommit:** Cluster with CPU overcommit

## Troubleshooting

### Issue: Pod in CrashLoopBackOff

**Cause:** Application crashing on startup or probe failing.

**Solution:**
1. Check logs: `kubectl logs <pod> -n <namespace> --previous`
2. Check events: `kubectl describe pod <pod> -n <namespace>`
3. Verify probe configuration
4. Verify environment variables and secrets

### Issue: Pod in Pending

**Cause:** Insufficient resources or incompatible node selector.

**Solution:**
1. Check pod events: `kubectl describe pod <pod>`
2. Check available resources: `kubectl top nodes`
3. Verify node affinity and tolerations
4. Scale node group if necessary

### Issue: Service not accessible

**Cause:** Incorrect selector, endpoints not ready, or network policy blocking.

**Solution:**
1. Check endpoints: `kubectl get endpoints <service>`
2. Verify pod labels match the selector
3. Check network policies
4. Test connectivity with debug pod

### Issue: High memory/CPU in pod

**Cause:** Memory leak, infinite loop, or excessive load.

**Solution:**
1. Check historical metrics in Grafana
2. Verify HPA is working: `kubectl get hpa`
3. Scale manually if urgent: `kubectl scale deployment <name> --replicas=<n>`
4. Investigate root cause in application logs

## Useful Commands

```bash
# List all pods in a namespace
kubectl get pods -n production

# View pod logs
kubectl logs -f <pod-name> -n production

# Execute shell in a pod
kubectl exec -it <pod-name> -n production -- /bin/sh

# View resource usage
kubectl top pods -n production

# Rollback deployment
kubectl rollout undo deployment/<name> -n production

# View deployment history
kubectl rollout history deployment/<name> -n production

# Port forward for local debug
kubectl port-forward svc/<service> 8080:80 -n production
```

## Related Links

- [Deploy Guide](../runbooks/deploy-guide.md) - How to deploy
- [Rollback Guide](../runbooks/rollback-guide.md) - Rollback procedure
- [Scaling Guide](../runbooks/scaling-guide.md) - How to scale services
- [Monitoring Stack](monitoring-stack.md) - Monitoring
- [System Overview](../architecture/system-overview.md) - Architecture
