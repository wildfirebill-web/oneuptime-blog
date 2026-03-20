# How to Monitor Harvester Cluster Health - Monitor

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Monitoring, Health

Description: Learn how to monitor the health of your Harvester cluster including nodes, VMs, storage, and network components using built-in and external tools.

## Introduction

Monitoring Harvester cluster health is essential for proactive incident detection and capacity planning. Harvester provides built-in health indicators in the dashboard, and also integrates with Prometheus and Grafana for detailed metrics and alerting. This guide covers key health checks, monitoring setup, and alerting configuration.

## Health Indicators in the Harvester Dashboard

The Harvester dashboard provides an at-a-glance health overview:

1. **Node Status**: Shows each node's state (Ready, NotReady, Maintenance)
2. **VM Status**: Running, Stopped, Errored VM counts
3. **Storage Health**: Longhorn volume health and utilization
4. **Network Status**: Load balancer and network attachment health
5. **Alerts**: Active alerts from the monitoring stack

## Step 1: CLI Health Checks

```bash
# Set kubeconfig

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

# ===== NODE HEALTH =====
echo "=== Node Status ==="
kubectl get nodes -o wide

echo "=== Node Resource Utilization ==="
kubectl top nodes

# Check for any node conditions (MemoryPressure, DiskPressure, etc.)
kubectl get nodes -o json | jq '.items[] | {
    name: .metadata.name,
    conditions: [.status.conditions[] | select(.status == "True") | .type]
}'

# ===== CONTROL PLANE HEALTH =====
echo "=== etcd Health ==="
kubectl exec -n kube-system \
    $(kubectl get pods -n kube-system -l component=etcd -o name | head -1) -- \
    etcdctl endpoint health \
    --cacert /var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
    --cert /var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
    --key /var/lib/rancher/rke2/server/tls/etcd/server-client.key \
    --cluster

# ===== VM HEALTH =====
echo "=== VM Instance Status ==="
kubectl get vmi -A -o custom-columns=\
'NAME:.metadata.name,NS:.metadata.namespace,PHASE:.status.phase,NODE:.status.nodeName'

# Count VMs by state
echo "=== VM Count by State ==="
kubectl get vmi -A -o json | jq '[.items[].status.phase] | group_by(.) |
    map({state: .[0], count: length})'

# ===== STORAGE HEALTH =====
echo "=== Longhorn Volume Health ==="
kubectl get volumes.longhorn.io -n longhorn-system \
    -o custom-columns=\
'NAME:.metadata.name,STATE:.status.state,ROBUSTNESS:.status.robustness,REPLICAS:.spec.numberOfReplicas'

# Check for degraded or faulted volumes
echo "=== Degraded/Unhealthy Volumes ==="
kubectl get volumes.longhorn.io -n longhorn-system \
    -o json | jq '.items[] | select(.status.robustness != "healthy") |
    {name: .metadata.name, robustness: .status.robustness}'

# ===== STORAGE SPACE =====
echo "=== Node Storage Utilization ==="
kubectl get nodes.longhorn.io -n longhorn-system \
    -o custom-columns=\
'NODE:.metadata.name,AVAILABLE:.status.diskStatus.default-disk-1.storageAvailable,MAX:.status.diskStatus.default-disk-1.storageMaximum'
```

## Step 2: Create a Health Check Script

```bash
#!/bin/bash
# harvester-health-check.sh - Comprehensive cluster health check

export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
ERRORS=0
WARNINGS=0

check_pass() { echo "  [PASS] $1"; }
check_warn() { echo "  [WARN] $1"; ((WARNINGS++)); }
check_fail() { echo "  [FAIL] $1"; ((ERRORS++)); }

echo "=================================================="
echo " Harvester Cluster Health Check - $(date)"
echo "=================================================="

# Check nodes
echo ""
echo "--- NODES ---"
TOTAL_NODES=$(kubectl get nodes --no-headers | wc -l)
READY_NODES=$(kubectl get nodes --no-headers | grep -c " Ready")
NOT_READY_NODES=$((TOTAL_NODES - READY_NODES))

[ $NOT_READY_NODES -eq 0 ] && check_pass "All ${TOTAL_NODES} nodes are Ready" || \
    check_fail "${NOT_READY_NODES}/${TOTAL_NODES} nodes are NOT Ready"

# Check for node pressure
PRESSURE_NODES=$(kubectl get nodes -o json | \
    jq -r '.items[] | select(.status.conditions[] |
    select(.type != "Ready" and .status == "True")) | .metadata.name' | \
    wc -l)
[ $PRESSURE_NODES -eq 0 ] && check_pass "No nodes under pressure" || \
    check_warn "${PRESSURE_NODES} node(s) have pressure conditions"

# Check etcd
echo ""
echo "--- ETCD ---"
ETCD_HEALTHY=$(kubectl get pods -n kube-system -l component=etcd \
    --no-headers | grep -c Running)
ETCD_TOTAL=$(kubectl get nodes --no-headers | wc -l)
[ $ETCD_HEALTHY -eq $ETCD_TOTAL ] && check_pass "${ETCD_HEALTHY}/${ETCD_TOTAL} etcd pods running" || \
    check_fail "Only ${ETCD_HEALTHY}/${ETCD_TOTAL} etcd pods running"

# Check system pods
echo ""
echo "--- SYSTEM PODS ---"
FAILING_PODS=$(kubectl get pods -n harvester-system --no-headers | \
    grep -v Running | grep -v Completed | wc -l)
[ $FAILING_PODS -eq 0 ] && check_pass "All harvester-system pods running" || \
    check_fail "${FAILING_PODS} pods not running in harvester-system"

LONGHORN_FAILING=$(kubectl get pods -n longhorn-system --no-headers | \
    grep -v Running | grep -v Completed | wc -l)
[ $LONGHORN_FAILING -eq 0 ] && check_pass "All longhorn-system pods running" || \
    check_fail "${LONGHORN_FAILING} pods not running in longhorn-system"

# Check volumes
echo ""
echo "--- STORAGE ---"
DEGRADED_VOLS=$(kubectl get volumes.longhorn.io -n longhorn-system \
    --no-headers | grep -v healthy | wc -l)
TOTAL_VOLS=$(kubectl get volumes.longhorn.io -n longhorn-system \
    --no-headers | wc -l)
[ $DEGRADED_VOLS -eq 0 ] && check_pass "${TOTAL_VOLS} volumes healthy" || \
    check_warn "${DEGRADED_VOLS}/${TOTAL_VOLS} volumes degraded"

# Check VMs
echo ""
echo "--- VIRTUAL MACHINES ---"
RUNNING_VMS=$(kubectl get vmi -A --no-headers | grep -c Running)
TOTAL_VMS=$(kubectl get vmi -A --no-headers | wc -l)
FAILED_VMS=$(kubectl get vmi -A --no-headers | grep -c -E "Failed|Unknown" || true)
check_pass "${RUNNING_VMS}/${TOTAL_VMS} VMs running"
[ $FAILED_VMS -eq 0 ] || check_fail "${FAILED_VMS} VMs in Failed/Unknown state"

echo ""
echo "=================================================="
echo " Summary: ${ERRORS} errors, ${WARNINGS} warnings"
echo "=================================================="
exit $ERRORS
```

```bash
chmod +x harvester-health-check.sh

# Run the health check
./harvester-health-check.sh

# Schedule as a cron job
echo "*/15 * * * * /opt/scripts/harvester-health-check.sh >> /var/log/harvester-health.log 2>&1" | \
    crontab -
```

## Step 3: Key Metrics to Monitor

Monitor these Prometheus metrics for proactive alerts:

```promql
# Node CPU utilization (alert if > 80%)
100 - (avg by (node) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Node memory utilization (alert if > 90%)
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Longhorn volume degraded (alert if > 0)
longhorn_volume_robustness == 2  # 2 = degraded, 3 = faulted

# Storage free space < 20%
longhorn_node_storage_usage_bytes / longhorn_node_storage_capacity_bytes > 0.8

# VM in error state
count by (namespace) (kubevirt_vmi_info{phase="Failed"}) > 0
```

## Conclusion

Regular health monitoring of your Harvester cluster prevents small issues from becoming critical failures. The combination of CLI-based health checks (for quick assessment), the built-in Harvester dashboard (for operational visibility), and Prometheus/Grafana (for trend analysis and alerting) provides comprehensive coverage. Automate the health check script and set up alerts for the critical metrics so your team is notified before users are impacted. Pay particular attention to Longhorn volume health and storage utilization, as degraded storage is often the leading indicator of upcoming problems.
