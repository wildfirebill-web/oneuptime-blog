# How to Set Up On-Call Procedures for Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, On-Call, Incident Response, SRE, Runbook, Alert, Escalation

Description: Define effective on-call procedures for Ceph storage operations including alert triage, escalation paths, runbook organization, and post-incident practices.

---

## Why Ceph On-Call Procedures Matter

Ceph cluster failures at 2 AM can be stressful without clear procedures. A well-defined on-call process ensures the engineer on duty knows exactly what to check, who to escalate to, and how to respond to each alert type. This reduces time-to-resolution and prevents mistakes made under pressure.

## Setting Up Alert Routing

Configure Grafana Alerting to route Ceph alerts by severity:

```yaml
# Grafana notification policy (via API or UI)
# Route critical alerts to PagerDuty, warnings to Slack

# Critical: pager
route:
  receiver: ops-team-default
  routes:
    - matchers:
        - alertname =~ "CephOSDDown|CephMonitorQuorumLost|CephClusterErrorState"
        - severity = critical
      receiver: pagerduty-ceph
      continue: false
    - matchers:
        - severity = warning
      receiver: slack-ceph-alerts
```

## On-Call Response Runbook: OSD Down Alert

```markdown
## Runbook: CephOSDDown

**Severity**: Warning (1 OSD down) / Critical (2+ OSDs down)
**SLO Impact**: Potential degraded redundancy; no client impact with 1 OSD down

### Step 1: Assess Scope
```

```bash
# How many OSDs are down?
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd stat

# Which specific OSDs are down?
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd tree | grep down

# Is the OSD pod running?
kubectl get pods -n rook-ceph | grep osd
```

```markdown
### Step 2: Prevent Data Movement (if OSD is temporarily down)
```

```bash
# Set noout to prevent unnecessary rebalancing
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd set noout
```

```markdown
### Step 3: Investigate
```

```bash
# Check OSD pod logs
OSD_POD=$(kubectl get pods -n rook-ceph | grep "osd-<id>" | awk '{print $1}')
kubectl logs -n rook-ceph ${OSD_POD} --tail=100

# Check node events
kubectl describe node <node-name> | grep -A20 Events
```

```markdown
### Step 4: Resolution Paths
- **OSD pod crashed**: Restart the pod and verify it comes back
- **Node is down**: Wait for node recovery or failover, then unset noout
- **Disk failure**: Replace drive, follow disk replacement runbook
- **Network issue**: Check node connectivity, resolve network issue

### Step 5: Recovery Verification
```

```bash
# Confirm OSD is back
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd stat

# Unset noout
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd unset noout

# Watch recovery
watch -n10 "kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph pg stat"
```

## Escalation Matrix

```markdown
## Escalation Paths for Ceph Incidents

| Situation | First Response | Escalation (15 min) | Critical Escalation (30 min) |
|-----------|---------------|---------------------|------------------------------|
| 1 OSD down | On-call SRE | Senior SRE | Storage architect |
| 2+ OSDs down | On-call SRE | Senior SRE + Storage lead | CTO/VP Eng |
| Monitor quorum lost | On-call SRE | Senior SRE + Storage lead | Storage architect |
| Cluster HEALTH_ERR | On-call SRE | All SREs on-call | Executive notification |
| Data loss suspected | Immediately escalate | - | - |
```

## On-Call Handoff Template

Use this for shift handoffs:

```markdown
## On-Call Handoff: $(date)

### Current Cluster Status
- Health: HEALTH_OK / HEALTH_WARN / HEALTH_ERR
- OSDs: X/Y up
- Active alerts: [list]

### Open Issues
- [Issue 1]: Status, owner, expected resolution
- [Issue 2]: Status, owner, expected resolution

### Scheduled Maintenance
- [Maintenance 1]: Date/time, scope, risk

### Recent Changes (last 24h)
- [Change 1]: What was changed, by whom

### Things to Watch
- [Watch item 1]: What to monitor and why
```

## Post-Incident Review Template

```markdown
## Post-Incident Review: [Incident Title]

**Date**: 2026-03-31
**Duration**: X hours Y minutes
**Severity**: P1 / P2 / P3
**On-call**: [name]

### Timeline
- HH:MM - Alert fired
- HH:MM - Engineer acknowledged
- HH:MM - Root cause identified
- HH:MM - Resolution implemented
- HH:MM - All clear

### Root Cause
[Technical description of what failed and why]

### What Went Well
- [Item 1]

### What Could Improve
- [Item 1]

### Action Items
- [ ] [Action 1] - Owner - Due date
```

## Summary

Effective Ceph on-call procedures combine clear alert routing by severity, step-by-step runbooks for each alert type, a defined escalation matrix, and structured shift handoffs. Storing runbooks in a shared wiki ensures any engineer on call can follow the same proven steps. Post-incident reviews capture lessons learned and drive continuous improvement of both the system and the on-call process.
