# Incident Response

## Overview

This guide describes TechCorp's incident response process. The goal is to restore services quickly, minimize impact to users, and learn from each incident to prevent recurrences.

## Incident Severities

| Severity | Description | Response Time | Examples |
|----------|-------------|---------------|----------|
| **SEV1** | System completely unavailable | 15 minutes | Site down, checkout broken |
| **SEV2** | Critical functionality degraded | 30 minutes | Payments failing, slow login |
| **SEV3** | Non-critical functionality affected | 2 hours | Slow search, delayed notifications |
| **SEV4** | Minor issue, no user impact | 24 hours | Excessive logging, false alert |

## Response Process

### Phase 1: Detection and Triage (0-15 min)

#### 1.1 Identify the Incident

Incidents can be detected by:
- Automatic alerts (PagerDuty, Alertmanager)
- User reports (Support, Social Media)
- Manual monitoring (Grafana, Logs)

#### 1.2 Create Incident Channel

```
/incident create "Brief problem description" --severity SEV2
```

This automatically creates:
- Slack channel #incident-YYYY-MM-DD-NNNN
- Ticket in tracking system
- Status page (if SEV1/SEV2)

#### 1.3 Escalate if Necessary

| Severity | Escalate to |
|----------|-------------|
| SEV1 | Tech Lead + Engineering Manager + CTO |
| SEV2 | Tech Lead + Engineering Manager |
| SEV3 | Tech Lead |
| SEV4 | On-call team |

### Phase 2: Diagnosis (15-45 min)

#### 2.1 Collect Information

```bash
# Check service status
kubectl get pods -n production

# Check recent metrics
# Access Grafana: https://grafana.techcorp.internal

# Check logs
kubectl logs -n production -l app=<suspect> --since=30m

# Check recent deploys
argocd app history <app-name>
```

#### 2.2 Identify Root Cause

Key questions:
- When did the problem start?
- Was there a recent deploy?
- Was there a configuration change?
- Is the problem in a specific service or general?
- Does the problem affect all users or a subset?

#### 2.3 Communicate Status

Update the incident channel every 15 minutes:

```
**Update [10:30]**
- Status: Investigating
- Hypothesis: Possible problem in payment-service after deploy
- Next steps: Checking logs and metrics
- ETA for resolution: Still undetermined
```

### Phase 3: Mitigation (Variable)

#### Option A: Rollback Deploy

If problem started after deploy:

```bash
argocd app rollback <app-name> <revision-id>
```

See: [Rollback Guide](rollback-guide.md)

#### Option B: Scale Resources

If problem is capacity-related:

```bash
kubectl scale deployment <app-name> -n production --replicas=10
```

See: [Scaling Guide](scaling-guide.md)

#### Option C: Disable Feature

If specific feature is causing problem:

```bash
curl -X POST https://feature-flags.techcorp.internal/api/flags/<feature>/disable
```

#### Option D: Failover

If problem is infrastructure-related:

```bash
# Execute database failover
aws rds failover-db-cluster --db-cluster-identifier techcorp-production

# Redirect traffic
kubectl patch ingress <app-name> -n production --patch '...'
```

### Phase 4: Resolution

#### 4.1 Confirm Resolution

- [ ] Metrics returned to normal
- [ ] Logs show no errors
- [ ] Manual tests passing
- [ ] No active alerts

#### 4.2 Communicate Resolution

```
**Incident Resolved [11:15]**
- Cause: Deploy with bug in payment-service
- Resolution: Rollback to previous version
- Duration: 45 minutes
- Impact: ~5% of payment transactions failed
```

#### 4.3 Update Status Page

For SEV1/SEV2, update status page:

```
/statuspage update "Services restored. Investigating root cause."
```

### Phase 5: Post-Mortem

For SEV1, SEV2, and recurring SEV3 incidents:

#### 5.1 Schedule Post-Mortem

- Within 48 hours for SEV1/SEV2
- Within 1 week for SEV3

#### 5.2 Post-Mortem Template

```markdown
# Post-Mortem: [Incident Title]

## Summary
[Brief incident description]

## Timeline
- [HH:MM] Event
- [HH:MM] Event
- [HH:MM] Incident resolved

## Impact
- Duration: X minutes/hours
- Users affected: X%
- Lost transactions: X

## Root Cause
[Detailed technical analysis]

## Resolution
[What was done to resolve]

## Lessons Learned
- What worked well
- What didn't work well
- Where we got lucky

## Action Items
- [ ] [Action] - [Owner] - [Deadline]
- [ ] [Action] - [Owner] - [Deadline]
```

## Runbooks by Incident Type

### Service Unavailable (503)

```bash
# 1. Check pods
kubectl get pods -n production -l app=<service>

# 2. If pods in CrashLoopBackOff
kubectl logs <pod> -n production --previous

# 3. If pods pending
kubectl describe pod <pod> -n production

# 4. If pods healthy but 503
# Check service and ingress
kubectl get svc,ingress -n production | grep <service>
```

### High Latency

```bash
# 1. Check latency metrics
# Grafana > Dashboard > Service Health

# 2. Check dependencies
# Grafana > Dashboard > Dependencies

# 3. If database slow
# See PostgreSQL metrics

# 4. If cache slow
# See Redis metrics
```

### High Error Rate

```bash
# 1. Identify errors in logs
kubectl logs -n production -l app=<service> --since=10m | grep -i error

# 2. Check if specific to endpoint
# Grafana > Dashboard > API Errors by Endpoint

# 3. Check external dependencies
curl -I https://api.external-provider.com/health
```

### Database Unavailable

```bash
# 1. Check status in RDS
aws rds describe-db-instances --db-instance-identifier techcorp-primary

# 2. Check connections
kubectl exec -it <pod> -n production -- psql -c "SELECT count(*) FROM pg_stat_activity"

# 3. Failover if necessary
aws rds failover-db-cluster --db-cluster-identifier techcorp-production
```

## Emergency Contacts

| Role | Name | Contact |
|------|------|---------|
| Engineering Manager | [Name] | [Phone] |
| Tech Lead Platform | [Name] | [Phone] |
| On-call DBA | [Name] | [Phone] |
| Security | [Name] | [Phone] |

## Related Links

- [Deploy Guide](deploy-guide.md) - Deploying fixes
- [Rollback Guide](rollback-guide.md) - Rolling back versions
- [Scaling Guide](scaling-guide.md) - Scaling resources
- [Monitoring Stack](../components/monitoring-stack.md) - Monitoring tools
