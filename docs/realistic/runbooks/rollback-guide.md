# Rollback Guide

## Overview

This guide describes the procedures for rolling back an application to a previous version when a problem is identified after deploy. Rollback should be considered when critical issues are detected and cannot be fixed quickly.

## When to Rollback

Consider rolling back when:

- Error rate increased significantly after deploy
- P99 latency is above acceptable levels
- Critical functionality is broken
- High severity incident was opened

**Do not automatically rollback if:**
- The problem can be resolved with a hotfix in less than 30 minutes
- The problem affects only a small percentage of users
- Rolling back would cause data loss

## Rollback Methods

### Method 1: Rollback via ArgoCD (Recommended)

This is the safest and most traceable method.

#### Step 1: Identify the Previous Version

```bash
# List deploy history
argocd app history <app-name>

# Example output:
# ID  DATE                           REVISION
# 1   2024-01-15T10:00:00Z          abc123 (v1.1.0)
# 2   2024-01-16T14:00:00Z          def456 (v1.2.0)  <- current with problem
```

#### Step 2: Execute the Rollback

```bash
# Rollback to specific revision
argocd app rollback <app-name> <ID>

# Example: rollback to ID 1
argocd app rollback my-service 1

# Check status
argocd app get <app-name>
```

#### Step 3: Confirm the Rollback

```bash
# Check pods
kubectl get pods -n production -l app=<app-name>

# Check image version
kubectl get deployment <app-name> -n production -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### Method 2: Rollback via Argo Rollouts

If using Argo Rollouts for canary or blue-green:

```bash
# Abort current rollout and rollback
kubectl argo rollouts abort <app-name> -n production

# Or rollback to specific revision
kubectl argo rollouts undo <app-name> -n production --to-revision=<revision>

# Monitor
kubectl argo rollouts get rollout <app-name> -n production --watch
```

### Method 3: Rollback via Git (Last Resort)

Use this method if the previous ones don't work.

```bash
# Identify stable version commit
git log --oneline -10

# Create rollback branch
git checkout -b revert/v1.2.0 main

# Revert problematic commit
git revert <commit-hash>

# Push and create emergency PR
git push origin revert/v1.2.0
gh pr create --base main --title "URGENT: Revert v1.2.0"
```

## Complete Rollback Procedure

### Phase 1: Detection and Decision (5 minutes)

1. **Confirm the problem**
   - Check metrics in Grafana
   - Check logs in Kibana
   - Confirm problem started after deploy

2. **Communicate with team**
   ```
   @here Problem detected after deploy of <app-name>
   Starting rollback procedure
   ```

3. **Decide to rollback**
   - If critical problem → rollback immediately
   - If moderate problem → evaluate hotfix vs rollback

### Phase 2: Rollback Execution (5-10 minutes)

1. **Execute rollback**
   ```bash
   argocd app rollback <app-name> <ID>
   ```

2. **Monitor progress**
   ```bash
   kubectl rollout status deployment/<app-name> -n production
   ```

3. **Verify health**
   ```bash
   kubectl get pods -n production -l app=<app-name>
   ```

### Phase 3: Validation (10-15 minutes)

1. **Check metrics**
   - Did error rate return to normal?
   - Did latency return to normal?
   - Did throughput return to normal?

2. **Run smoke tests**
   ```bash
   # Test critical endpoints
   curl -I https://api.techcorp.com/health
   curl -I https://api.techcorp.com/v1/products
   ```

3. **Check logs**
   ```bash
   kubectl logs -n production -l app=<app-name> --tail=100 | grep -i error
   ```

### Phase 4: Communication (5 minutes)

1. **Update status**
   ```
   @here Rollback of <app-name> completed
   Current version: v1.1.0
   Metrics normalized
   ```

2. **Create incident report**
   - What happened
   - Impact
   - Actions taken
   - Next steps

## Special Cases

### Rollback with Database Migrations

If the problematic version included migrations:

1. **Evaluate impact**
   - Is migration reversible?
   - Is there data loss?
   - Time needed to reverse migration?

2. **If migration is reversible:**
   ```bash
   # Execute rollback migration
   kubectl exec -it <pod> -n production -- ./migrate rollback

   # Then rollback application
   argocd app rollback <app-name> <ID>
   ```

3. **If migration is NOT reversible:**
   - Keep database at current version
   - Rollback only application (if compatible)
   - Or do hotfix to correct the problem

### Rollback Multiple Services

If the deploy affected multiple services:

```bash
# Rollback in reverse order of deploy
argocd app rollback service-c <ID>
argocd app rollback service-b <ID>
argocd app rollback service-a <ID>
```

### Rollback with Feature Flags

If the problematic feature uses feature flag:

```bash
# Faster option: disable feature
curl -X POST https://feature-flags.techcorp.internal/api/flags/new-checkout/disable

# Then evaluate if rollback is still necessary
```

## Rollback Checklist

### Before Rolling Back

- [ ] Confirm problem is caused by the deploy
- [ ] Identify stable previous version
- [ ] Communicate with team and stakeholders
- [ ] Check if migrations are involved

### During Rollback

- [ ] Execute rollback command
- [ ] Monitor rollout status
- [ ] Verify pods are healthy

### After Rollback

- [ ] Validate metrics normalized
- [ ] Run smoke tests
- [ ] Communicate rollback completion
- [ ] Document the incident
- [ ] Schedule post-mortem

## Preventing Need for Rollback

To reduce the need for rollbacks:

1. **Canary deploy** - Detect problems with partial traffic
2. **Feature flags** - Disable features without rolling back code
3. **Automated tests** - Detect problems before deploy
4. **Post-deploy smoke tests** - Detect problems immediately

## Related Links

- [Deploy Guide](deploy-guide.md) - Deploy process
- [Incident Response](incident-response.md) - Incident management
- [Kubernetes Cluster](../components/kubernetes-cluster.md) - K8s commands
- [Common Errors](../troubleshooting/common-errors.md) - Diagnosis
