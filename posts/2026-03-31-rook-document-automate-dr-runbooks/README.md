# How to Document and Automate Rook-Ceph DR Runbooks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Runbook, Automation, Disaster Recovery

Description: Learn how to create structured DR runbooks for Rook-Ceph and automate recovery steps using scripts and GitOps workflows.

---

A disaster recovery runbook documents the exact steps needed to recover from a specific failure scenario. For Rook-Ceph, good runbooks reduce recovery time, prevent mistakes under pressure, and enable team members who did not build the system to perform recovery. This guide covers runbook structure and automation.

## Runbook Structure

Every Rook-Ceph runbook should follow a consistent structure:

```text
# Runbook: [Failure Scenario Name]

## Metadata
- Severity: Critical / High / Medium
- RTO Target: [time]
- Last Tested: [date]
- Owner: [team/person]

## Detection
[How to detect this failure. Include monitoring alerts that trigger this runbook.]

## Impact
[What is broken. Which applications are affected.]

## Prerequisites
[What access, tools, and information are needed before starting recovery.]

## Steps
[Numbered, executable steps with exact commands.]

## Verification
[How to confirm recovery is complete.]

## Escalation
[When to escalate and to whom.]

## Post-Incident
[What to document and improve after recovery.]
```

## Example: OSD Failure Runbook

```markdown
# Runbook: Rook-Ceph OSD Failure

## Metadata
- Severity: High
- RTO Target: 30 minutes
- Last Tested: 2024-01-10
- Owner: storage-team

## Detection
Alert: ceph_osd_up == 0 for any OSD
Symptom: Applications report slow I/O

## Steps

### Step 1: Identify failed OSD
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree

### Step 2: Check pod status
kubectl -n rook-ceph get pods -l app=rook-ceph-osd -o wide

### Step 3: Check node status
kubectl get nodes

### Step 4: If node is healthy, restart OSD pod
kubectl -n rook-ceph delete pod <osd-pod-name>

### Step 5: If disk is failed, replace it
# Follow the OSD disk replacement procedure

## Verification
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
# Expected: HEALTH_OK, all OSDs up
```

## Storing Runbooks

Store runbooks in Git alongside your Rook configuration:

```text
rook-config/
  clusters/production/
    cephcluster.yaml
    pools/
    runbooks/
      osd-failure.md
      mon-quorum-loss.md
      namespace-deletion.md
      node-failure.md
      upgrade-rollback.md
```

Use pull requests for runbook changes and require review from someone who has performed the recovery. Git history provides an audit trail of what changed and why.

## Automating Common Recovery Steps

Convert repetitive runbook steps into scripts:

```bash
#!/bin/bash
# recover-failed-osd.sh

set -euo pipefail

NAMESPACE="${1:-rook-ceph}"
OSD_ID="${2:-}"

if [ -z "$OSD_ID" ]; then
  echo "Usage: $0 <namespace> <osd-id>"
  exit 1
fi

echo "=== Starting OSD recovery for osd.$OSD_ID ==="

echo "--- Current OSD status ---"
kubectl -n "$NAMESPACE" exec deploy/rook-ceph-tools -- \
  ceph osd tree | grep -A 2 "osd.$OSD_ID"

echo "--- Checking OSD pod ---"
kubectl -n "$NAMESPACE" get pods -l "ceph-osd-id=$OSD_ID" -o wide

echo "--- Restarting OSD pod ---"
kubectl -n "$NAMESPACE" delete pod -l "ceph-osd-id=$OSD_ID"

echo "--- Waiting for recovery ---"
kubectl -n "$NAMESPACE" wait pod \
  -l "ceph-osd-id=$OSD_ID" \
  --for=condition=Ready \
  --timeout=300s

echo "--- Verifying cluster health ---"
kubectl -n "$NAMESPACE" exec deploy/rook-ceph-tools -- ceph status
```

## GitOps-Driven DR with ArgoCD

Use ArgoCD to manage DR configuration as code. Create an ArgoCD Application that manages your Rook configuration and can restore it from Git:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rook-ceph-disaster-recovery
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/rook-config
    targetRevision: HEAD
    path: clusters/production
  destination:
    server: https://kubernetes.default.svc
    namespace: rook-ceph
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

With `selfHeal: true`, ArgoCD automatically restores deleted Rook resources from Git, dramatically reducing recovery time for accidental deletions.

## Runbook Validation

Validate runbooks periodically by executing them in staging:

```bash
#!/bin/bash
# validate-runbook.sh
# Runs runbook steps against staging cluster

RUNBOOK="$1"
STAGING_CONTEXT="kind-staging"

kubectl config use-context "$STAGING_CONTEXT"
echo "Running runbook validation: $RUNBOOK"
bash "$RUNBOOK" --dry-run 2>&1 | tee "validation-$(date +%Y%m%d).log"
```

## Summary

Effective Rook-Ceph DR runbooks follow a consistent structure covering detection, impact, prerequisites, numbered steps with exact commands, verification, and escalation paths. Store runbooks in Git alongside Rook configuration, automate repetitive recovery steps as scripts, and use GitOps tools like ArgoCD to automatically restore accidentally deleted resources. Regular validation of runbooks in staging ensures they remain accurate and effective when needed in production.
