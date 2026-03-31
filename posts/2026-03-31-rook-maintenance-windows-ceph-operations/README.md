# How to Plan Maintenance Windows for Ceph Operations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Maintenance, Window, Operations, Planning, Upgrade, OSD

Description: Plan and execute safe Ceph maintenance windows for OSD replacement, cluster upgrades, and configuration changes while minimizing impact on storage clients.

---

## Why Maintenance Windows Matter

Ceph maintenance activities - OSD replacement, node reboots, Ceph version upgrades, and network changes - require careful planning. Performing maintenance on a cluster that is already degraded, or failing to communicate downtime, leads to data unavailability and frustrated users. A structured maintenance window process reduces risk and builds trust.

## Choosing a Maintenance Window

Select maintenance windows based on:
- **Traffic patterns**: Use Grafana to identify the lowest-IO period (often 2-6 AM)
- **Backup completion**: Ensure nightly backups have finished
- **Change freeze windows**: Avoid maintenance during product launches or peak business periods
- **Team availability**: Have senior engineers available for the full window

## Pre-Maintenance Preparation (T-24 hours)

```bash
#!/bin/bash
# pre-maintenance-check.sh

echo "=== Pre-Maintenance Cluster Assessment ==="

# 1. Cluster must be HEALTH_OK
HEALTH=$(kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph health)
if [[ "$HEALTH" != "HEALTH_OK" ]]; then
    echo "FAIL: Cluster is not healthy: $HEALTH"
    echo "Do not proceed with maintenance until cluster is HEALTH_OK"
    exit 1
fi
echo "PASS: Cluster health: $HEALTH"

# 2. Verify no recovery in progress
DEGRADED=$(kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph pg stat | grep -c "degraded" || true)
if [[ "$DEGRADED" -gt 0 ]]; then
    echo "FAIL: Active recovery in progress - postpone maintenance"
    exit 1
fi
echo "PASS: No active recovery"

# 3. Check disk space headroom (need >30% free for recovery)
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph df | awk '/TOTAL/ {print "Cluster usage:", $6}'

echo "=== Pre-check complete. Safe to proceed. ==="
```

## Maintenance Window Execution: OSD Node Maintenance

Safely drain a node for maintenance:

```bash
# Step 1: Set maintenance flags
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd set noout

kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd set norebalance

# Step 2: Drain the Kubernetes node
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=300s

# Step 3: Verify OSDs on this node are stopped
kubectl get pods -n rook-ceph -o wide | grep <node-name>

# Step 4: Perform maintenance (OS patch, hardware work, etc.)
echo "Performing maintenance on <node-name>..."

# Step 5: Restore the node
kubectl uncordon <node-name>

# Step 6: Wait for OSD pods to restart
kubectl wait --for=condition=Ready \
  -l app=rook-ceph-osd \
  -n rook-ceph \
  pods --timeout=300s

# Step 7: Verify OSDs are back
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd stat

# Step 8: Remove maintenance flags
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd unset noout

kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd unset norebalance

# Step 9: Verify cluster health
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph health
```

## Configuring Grafana Silences for the Window

Prevent alert noise during planned maintenance:

```bash
# Create a Grafana silence via API
GRAFANA_URL="http://grafana.monitoring.svc"
START=$(date -u -v+5M +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
        date -u -d '+5 minutes' +%Y-%m-%dT%H:%M:%SZ)
END=$(date -u -v+4H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
      date -u -d '+4 hours' +%Y-%m-%dT%H:%M:%SZ)

curl -s -X POST "${GRAFANA_URL}/api/alertmanager/grafana/api/v2/silences" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${GRAFANA_TOKEN}" \
  -d "{
    \"matchers\": [{\"name\": \"namespace\", \"value\": \"rook-ceph\", \"isRegex\": false}],
    \"startsAt\": \"${START}\",
    \"endsAt\": \"${END}\",
    \"comment\": \"Planned maintenance window - node drain\",
    \"createdBy\": \"ops-team\"
  }"
```

## Maintenance Communication Template

```markdown
Subject: [STORAGE MAINTENANCE] Ceph Cluster - 2026-03-31 02:00-04:00 UTC

**What**: Planned Ceph OSD node maintenance (OS patching)
**When**: 2026-03-31 02:00-04:00 UTC
**Impact**: No client impact expected. Cluster redundancy temporarily reduced during maintenance.
**Risk**: Low - cluster is healthy with full redundancy before start.
**Rollback**: Re-uncordon node, unset maintenance flags.
**Contact**: #storage-ops Slack channel during window
```

## Post-Maintenance Verification

```bash
#!/bin/bash
# post-maintenance-check.sh

echo "=== Post-Maintenance Verification ==="

# Check cluster health
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph health detail

# Verify all OSDs are up
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph osd stat

# Check no maintenance flags are set
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd dump | grep -E "^flags"

# Watch recovery if cluster degraded
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph pg stat

echo "=== Post-maintenance check complete ==="
```

## Summary

Safe Ceph maintenance windows require pre-checks to confirm the cluster is healthy, maintenance flags (`noout`, `norebalance`) to prevent automatic data movement during the work, Grafana silences to suppress expected alerts, and structured post-maintenance verification. Communicating the window with scope and expected impact builds stakeholder trust. Scripting the common steps (drain, uncordon, flag management) ensures consistency regardless of which engineer performs the maintenance.
