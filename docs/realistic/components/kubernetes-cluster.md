# Kubernetes Cluster

## Descrição

O Kubernetes Cluster é a plataforma de orquestração de containers da TechCorp, responsável por executar todos os microsserviços em ambiente de produção. O cluster gerencia o ciclo de vida das aplicações, incluindo deployment, scaling, networking e self-healing.

A infraestrutura utiliza clusters gerenciados (EKS na AWS), permitindo que o time de plataforma foque na operação das aplicações ao invés da manutenção do control plane. O cluster é configurado com múltiplos node groups otimizados para diferentes tipos de workload.

A arquitetura implementa isolamento por namespaces, com ambientes separados para produção, staging e desenvolvimento. Políticas de rede (Network Policies) garantem que serviços só se comuniquem conforme permitido.

## Responsáveis

- **Time:** Platform Engineering
- **Tech Lead:** Roberto Nascimento
- **Slack:** #platform-k8s

## Stack Tecnológica

- Orquestração: Kubernetes 1.28 (EKS)
- Service Mesh: Istio 1.19
- Ingress: AWS ALB Ingress Controller
- Secrets: External Secrets Operator + AWS Secrets Manager
- GitOps: ArgoCD
- Monitoramento: Prometheus + Grafana

## Arquitetura do Cluster

### Node Groups

| Node Group | Tipo de Instância | Finalidade | Scaling |
|------------|------------------|------------|---------|
| `general` | m6i.xlarge | Workloads gerais | 3-10 |
| `compute` | c6i.2xlarge | CPU-intensive | 2-8 |
| `memory` | r6i.xlarge | Memory-intensive | 2-6 |
| `spot` | m6i.xlarge (spot) | Batch jobs | 0-20 |

### Namespaces

| Namespace | Descrição |
|-----------|-----------|
| `production` | Serviços de produção |
| `staging` | Ambiente de staging |
| `monitoring` | Prometheus, Grafana |
| `istio-system` | Service mesh |
| `argocd` | GitOps controller |
| `external-secrets` | Secrets operator |

## Configuração

### Acesso ao Cluster

```bash
# Configurar kubeconfig
aws eks update-kubeconfig --name techcorp-production --region us-east-1

# Verificar conexão
kubectl cluster-info

# Listar namespaces
kubectl get namespaces
```

### Contextos Disponíveis

```bash
# Produção
kubectl config use-context techcorp-production

# Staging
kubectl config use-context techcorp-staging
```

## Padrões de Deployment

### Estrutura de Manifests

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

### Resource Quotas por Namespace

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

## Monitoramento

- **Dashboard Grafana:** https://grafana.techcorp.internal/d/kubernetes
- **ArgoCD:** https://argocd.techcorp.internal
- **Métricas Prometheus:** https://prometheus.techcorp.internal

### Métricas Principais

| Métrica | Descrição | Alerta |
|---------|-----------|--------|
| `kube_pod_status_ready` | Pods em estado Ready | < expected |
| `kube_node_status_condition` | Saúde dos nodes | NotReady |
| `container_memory_usage_bytes` | Uso de memória | > 80% limit |
| `container_cpu_usage_seconds` | Uso de CPU | > 80% limit |

### Alertas Configurados

- **KubePodNotReady:** Pod não está Ready por mais de 5 minutos
- **KubeNodeNotReady:** Node não está Ready por mais de 5 minutos
- **KubePodCrashLooping:** Pod reiniciando repetidamente
- **KubeMemoryOvercommit:** Cluster com overcommit de memória
- **KubeCPUOvercommit:** Cluster com overcommit de CPU

## Troubleshooting

### Problema: Pod em CrashLoopBackOff

**Causa:** Aplicação crashando na inicialização ou probe falhando.

**Solução:**
1. Verificar logs: `kubectl logs <pod> -n <namespace> --previous`
2. Verificar eventos: `kubectl describe pod <pod> -n <namespace>`
3. Verificar configuração de probes
4. Verificar variáveis de ambiente e secrets

### Problema: Pod em Pending

**Causa:** Recursos insuficientes ou node selector incompatível.

**Solução:**
1. Verificar eventos do pod: `kubectl describe pod <pod>`
2. Verificar recursos disponíveis: `kubectl top nodes`
3. Verificar node affinity e tolerations
4. Escalar node group se necessário

### Problema: Service não acessível

**Causa:** Selector incorreto, endpoints não prontos ou network policy bloqueando.

**Solução:**
1. Verificar endpoints: `kubectl get endpoints <service>`
2. Verificar labels dos pods correspondem ao selector
3. Verificar network policies
4. Testar conectividade com pod de debug

### Problema: High memory/CPU em pod

**Causa:** Leak de memória, loop infinito ou carga excessiva.

**Solução:**
1. Verificar métricas históricas no Grafana
2. Verificar HPA está funcionando: `kubectl get hpa`
3. Escalar manualmente se urgente: `kubectl scale deployment <name> --replicas=<n>`
4. Investigar causa raiz nos logs da aplicação

## Comandos Úteis

```bash
# Listar todos os pods em um namespace
kubectl get pods -n production

# Ver logs de um pod
kubectl logs -f <pod-name> -n production

# Executar shell em um pod
kubectl exec -it <pod-name> -n production -- /bin/sh

# Ver uso de recursos
kubectl top pods -n production

# Rollback de deployment
kubectl rollout undo deployment/<name> -n production

# Ver histórico de deployments
kubectl rollout history deployment/<name> -n production

# Port forward para debug local
kubectl port-forward svc/<service> 8080:80 -n production
```

## Links Relacionados

- [Guia de Deploy](../runbooks/deploy-guide.md) - Como fazer deploy
- [Guia de Rollback](../runbooks/rollback-guide.md) - Procedimento de rollback
- [Guia de Escalar](../runbooks/scaling-guide.md) - Como escalar serviços
- [Monitoring Stack](monitoring-stack.md) - Monitoramento
- [Visão Geral da Arquitetura](../architecture/system-overview.md) - Arquitetura
