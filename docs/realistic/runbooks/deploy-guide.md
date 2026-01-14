# Deploy Guide

## Overview

This guide describes the standard process for deploying applications on the TechCorp platform. The process uses GitOps with ArgoCD for declarative application management on Kubernetes.

## Prerequisites

Before starting a deploy, ensure you have:

- [ ] Access to the application repository
- [ ] Permission in ArgoCD for the target environment
- [ ] CI pipeline passed successfully
- [ ] Code review approved
- [ ] Automated tests passing

## Environments

| Environment | Branch | Cluster | Auto-sync |
|-------------|--------|---------|-----------|
| Development | develop | techcorp-dev | Yes |
| Staging | release/* | techcorp-staging | Yes |
| Production | main | techcorp-production | No |

## Deploy Process

### 1. Deploy to Development

Deploy to development is automatic when merging to the `develop` branch.

```bash
# Create feature branch
git checkout -b feature/new-functionality

# Develop and commit
git add .
git commit -m "feat: add new functionality"

# Push and create PR to develop
git push origin feature/new-functionality
```

After the merge:
1. CI pipeline runs build and tests
2. Docker image is built and pushed to registry
3. ArgoCD detects the change and deploys automatically

### 2. Deploy to Staging

Deploy to staging is automatic when creating a `release/*` branch.

```bash
# Create release branch from develop
git checkout develop
git pull
git checkout -b release/1.2.0

# Push the release branch
git push origin release/1.2.0
```

ArgoCD automatically deploys to the staging environment.

### 3. Deploy to Production

Deploy to production requires manual approval and follows a controlled process.

#### Step 1: Prepare the Deploy

```bash
# Ensure release branch is up to date
git checkout release/1.2.0
git pull

# Create PR to main
# Via GitHub UI or gh CLI:
gh pr create --base main --head release/1.2.0 --title "Release 1.2.0"
```

#### Step 2: Approve the Deploy

1. Get PR approval (minimum 2 reviewers)
2. Verify all checks passed
3. Merge the PR

#### Step 3: Execute the Deploy

After the merge, deploy is NOT automatic in production. Execute manually:

```bash
# Access ArgoCD
# https://argocd.techcorp.internal

# Or via CLI:
argocd login argocd.techcorp.internal

# Sync application
argocd app sync <app-name> --prune

# Check status
argocd app get <app-name>
```

#### Step 4: Verify the Deploy

```bash
# Check pods
kubectl get pods -n production -l app=<app-name>

# Check logs
kubectl logs -n production -l app=<app-name> --tail=100

# Check metrics
# Access Grafana: https://grafana.techcorp.internal/d/<app-name>
```

## Canary Deploy

For high-risk deploys, use the canary strategy:

### Configuration

```yaml
# values.yaml
canary:
  enabled: true
  steps:
    - setWeight: 10
    - pause: { duration: 5m }
    - setWeight: 30
    - pause: { duration: 5m }
    - setWeight: 50
    - pause: { duration: 10m }
    - setWeight: 100
  analysis:
    successCondition: result[0] >= 0.95
    failureLimit: 3
```

### Execution

```bash
# Start canary
argocd app sync <app-name> --strategy canary

# Monitor progress
kubectl argo rollouts get rollout <app-name> -n production --watch

# Promote manually (if paused)
kubectl argo rollouts promote <app-name> -n production

# Abort if necessary
kubectl argo rollouts abort <app-name> -n production
```

## Blue-Green Deploy

For deploys that require instant rollback:

### Configuration

```yaml
# values.yaml
blueGreen:
  enabled: true
  autoPromotionEnabled: false
  previewService: <app-name>-preview
  activeService: <app-name>
```

### Execution

```bash
# Deploy (creates preview version)
argocd app sync <app-name>

# Test preview
curl https://<app-name>-preview.techcorp.internal/health

# Promote to active
kubectl argo rollouts promote <app-name> -n production
```

## Deploy Checklist

### Before Deploy

- [ ] Code reviewed and approved
- [ ] Automated tests passing
- [ ] Documentation updated (if necessary)
- [ ] Database migrations prepared
- [ ] Feature flags configured (if applicable)
- [ ] Communication with stakeholders

### During Deploy

- [ ] Monitor Grafana dashboard
- [ ] Check error logs
- [ ] Test critical endpoints
- [ ] Check latency metrics

### After Deploy

- [ ] Verify all pods are healthy
- [ ] Confirm metrics are normal
- [ ] Run smoke tests
- [ ] Update status in #releases channel

## Troubleshooting

### Deploy stuck in "Progressing"

**Cause:** Pods cannot start or probes failing.

**Solution:**
1. Check events: `kubectl describe pod <pod> -n production`
2. Check logs: `kubectl logs <pod> -n production`
3. Check resources: `kubectl top pods -n production`

### Image not found

**Cause:** CI pipeline did not complete or incorrect tag.

**Solution:**
1. Check pipeline in CI
2. Confirm image tag in registry
3. Check `image` in deployment.yaml

### Secrets not available

**Cause:** External Secrets Operator did not sync.

**Solution:**
1. Check ExternalSecret: `kubectl get externalsecrets -n production`
2. Check operator logs
3. Force sync if necessary

### ArgoCD shows OutOfSync

**Cause:** Difference between desired and actual state.

**Solution:**
1. Check diff: `argocd app diff <app-name>`
2. If expected, sync: `argocd app sync <app-name>`
3. If not expected, investigate manual changes

## Recommended Times

| Deploy Type | Allowed Time | Restrictions |
|-------------|--------------|--------------|
| Critical hotfix | Any | Tech Lead approval |
| Normal deploy | 09:00 - 16:00 (weekdays) | Not on Fridays after 14:00 |
| Deploy with migration | 09:00 - 12:00 (weekdays) | Not on Fridays |

## Related Links

- [Rollback Guide](rollback-guide.md) - How to rollback a deploy
- [Incident Response](incident-response.md) - If something goes wrong
- [Kubernetes Cluster](../components/kubernetes-cluster.md) - Infrastructure
- [System Overview](../architecture/system-overview.md) - Architecture
