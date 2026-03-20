# How to Perform a Rolling Upgrade of RKE2

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Rolling Upgrade, Zero Downtime, Rancher

Description: Learn how to perform a zero-downtime rolling upgrade of your RKE2 cluster, upgrading nodes one at a time while keeping workloads running.

A rolling upgrade minimizes downtime by upgrading one node at a time, draining workloads before the upgrade and restoring them afterward. This is the recommended approach for production RKE2 clusters where continuous availability is required. This guide provides a detailed, step-by-step process for a safe rolling upgrade.

## Prerequisites

- An operational RKE2 cluster with multiple nodes
- An etcd backup taken before starting
- Pod Disruption Budgets (PDBs) configured for critical applications
- Rancher or kubectl access

## Pre-Upgrade Steps

```bash
# STEP 1: Take an etcd snapshot backup

sudo rke2 etcd-snapshot save \
  --name rolling-upgrade-backup-$(date +%Y%m%d)

echo "Backup created:"
ls -lh /var/lib/rancher/rke2/server/db/snapshots/ | tail -5

# STEP 2: Verify cluster health
echo "Checking cluster health..."
kubectl get nodes
kubectl get pods -A | grep -v -E "Running|Completed"

# STEP 3: Check PDBs for critical applications
kubectl get pdb -A

# STEP 4: Identify all nodes and their roles
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.conditions[-1].type,\
ROLE:.metadata.labels."node-role\.kubernetes\.io/control-plane",\
VERSION:.status.nodeInfo.kubeletVersion
```

## Phase 1: Upgrade Control Plane Nodes

Control plane nodes must be upgraded first. With multiple server nodes, upgrade one at a time:

```bash
# Script for rolling upgrade of server nodes
upgrade_server_node() {
  local NODE=$1
  local VERSION=$2

  echo "=== Upgrading server node: $NODE ==="

  # Step 1: Cordon the node to prevent new pods scheduling
  echo "Cordoning $NODE..."
  kubectl cordon $NODE

  # Step 2: Wait for any running workloads to complete
  # Note: Control plane pods cannot be evicted
  echo "Waiting for non-system pods..."
  sleep 30

  # Step 3: SSH to the node and upgrade RKE2
  echo "Installing RKE2 $VERSION on $NODE..."
  ssh $NODE "curl -sfL https://get.rke2.io | \
    INSTALL_RKE2_VERSION=$VERSION sudo sh -"

  # Step 4: Restart the RKE2 server service
  echo "Restarting RKE2 server on $NODE..."
  ssh $NODE "sudo systemctl restart rke2-server"

  # Step 5: Wait for the node to come back online
  echo "Waiting for $NODE to be ready..."
  kubectl wait node/$NODE --for=condition=Ready --timeout=300s

  # Step 6: Verify the node version
  NEW_VER=$(kubectl get node $NODE \
    -o jsonpath='{.status.nodeInfo.kubeletVersion}')
  echo "Node $NODE is now running: $NEW_VER"

  # Step 7: Uncordon the node
  kubectl uncordon $NODE

  # Step 8: Allow time for recovery before next node
  echo "Waiting 60 seconds before next node..."
  sleep 60

  echo "=== Server node $NODE upgrade complete ==="
}

# Upgrade each server node
TARGET_VERSION="v1.28.10+rke2r1"
upgrade_server_node "server-node-1" "$TARGET_VERSION"
upgrade_server_node "server-node-2" "$TARGET_VERSION"
upgrade_server_node "server-node-3" "$TARGET_VERSION"
```

## Phase 2: Upgrade Worker Nodes

After all server nodes are upgraded, upgrade worker nodes:

```bash
# Script for rolling upgrade of worker nodes
upgrade_worker_node() {
  local NODE=$1
  local VERSION=$2

  echo "=== Upgrading worker node: $NODE ==="

  # Step 1: Drain the node (evict all pods)
  echo "Draining $NODE..."
  kubectl drain $NODE \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --grace-period=30 \
    --timeout=120s

  # Step 2: Verify the drain was successful
  POD_COUNT=$(kubectl get pods -A \
    --field-selector spec.nodeName=$NODE \
    --no-headers | grep -v daemonset | wc -l)
  echo "Remaining pods on $NODE (daemonsets excluded): $POD_COUNT"

  # Step 3: SSH to the node and upgrade RKE2 agent
  echo "Installing RKE2 agent $VERSION on $NODE..."
  ssh $NODE "curl -sfL https://get.rke2.io | \
    INSTALL_RKE2_VERSION=$VERSION \
    INSTALL_RKE2_TYPE=agent sudo sh -"

  # Step 4: Restart the RKE2 agent
  echo "Restarting RKE2 agent on $NODE..."
  ssh $NODE "sudo systemctl restart rke2-agent"

  # Step 5: Wait for node to be ready
  echo "Waiting for $NODE to rejoin cluster..."
  kubectl wait node/$NODE --for=condition=Ready --timeout=300s

  # Step 6: Verify version
  NEW_VER=$(kubectl get node $NODE \
    -o jsonpath='{.status.nodeInfo.kubeletVersion}')
  echo "Node $NODE is now running: $NEW_VER"

  # Step 7: Uncordon the node
  kubectl uncordon $NODE

  # Step 8: Verify workloads are scheduling on the node
  echo "Verifying workloads are running on $NODE..."
  sleep 30
  kubectl get pods -o wide -A | grep $NODE | grep Running | head -5

  echo "=== Worker node $NODE upgrade complete ==="
}

# Upgrade workers with health checks between each node
TARGET_VERSION="v1.28.10+rke2r1"
WORKERS=("worker-node-1" "worker-node-2" "worker-node-3" "worker-node-4")

for worker in "${WORKERS[@]}"; do
  # Check cluster health before each node upgrade
  echo "Checking cluster health before upgrading $worker..."
  NOT_READY=$(kubectl get nodes --no-headers | grep -v Ready | wc -l)
  if [ "$NOT_READY" -gt 0 ]; then
    echo "WARNING: Some nodes are not ready. Waiting..."
    kubectl wait nodes --all --for=condition=Ready --timeout=300s
  fi

  upgrade_worker_node "$worker" "$TARGET_VERSION"
done
```

## Monitor Rolling Upgrade Progress

```bash
# Watch node versions and status in real-time
watch -n 5 'kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.conditions[-1].type,\
VERSION:.status.nodeInfo.kubeletVersion'

# Monitor pod distribution during upgrade
watch -n 10 'kubectl get pods -A -o wide | grep -v kube-system | \
  awk "{print \$8}" | sort | uniq -c | sort -rn'

# Check for any pods stuck in pending due to drain
kubectl get pods -A | grep -E "Pending|Evicted|Terminating"
```

## Post-Upgrade Validation

```bash
# Verify all nodes are on the target version
TARGET_VERSION="v1.28.10+rke2r1"
echo "=== Version Verification ==="
kubectl get nodes -o custom-columns=\
  NAME:.metadata.name,\
  VERSION:.status.nodeInfo.kubeletVersion

# Check if any nodes are on the old version
OLD_COUNT=$(kubectl get nodes \
  -o jsonpath='{.items[*].status.nodeInfo.kubeletVersion}' | \
  tr ' ' '\n' | grep -v "$TARGET_VERSION" | wc -l)

if [ "$OLD_COUNT" -eq 0 ]; then
  echo "All nodes successfully upgraded to $TARGET_VERSION"
else
  echo "WARNING: $OLD_COUNT nodes are still on old version"
fi

# Final health check
echo "=== Final Health Check ==="
kubectl get pods -A | grep -v -E "Running|Completed|Succeeded"
echo "Number of non-running pods: $(kubectl get pods -A | grep -v -E "Running|Completed|Succeeded" | grep -v NAMESPACE | wc -l)"
```

## Conclusion

A rolling upgrade of RKE2 requires patience and careful execution, but ensures zero downtime for your workloads. The key principles are: upgrade server nodes first (one at a time), drain worker nodes before upgrading, verify each node is healthy before moving to the next, and validate the entire cluster after completion. For larger clusters, consider using the system-upgrade-controller to automate this process with built-in health checks and concurrency controls.
