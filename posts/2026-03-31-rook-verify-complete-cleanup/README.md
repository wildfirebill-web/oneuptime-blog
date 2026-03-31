# How to Verify Complete Rook-Ceph Cleanup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Cleanup, Verification, Uninstall

Description: Learn how to verify that a Rook-Ceph cluster has been completely removed from Kubernetes and storage nodes, ensuring a clean state for reinstallation.

---

After removing a Rook-Ceph cluster, it is essential to verify that all components have been cleaned up before declaring the process complete or attempting a reinstallation. Incomplete cleanup is the most common cause of failed Rook reinstallations. This guide provides a comprehensive verification checklist.

## Layer 1: Kubernetes Resources

Start by verifying all Kubernetes-level resources are removed.

### Namespace

```bash
kubectl get namespace rook-ceph
```

Expected: `Error from server (NotFound)` or empty output.

### Custom Resources

```bash
kubectl get crd | grep -E "rook|ceph"
```

Expected: No output (all Rook CRDs removed).

### Cluster-Scoped Resources

```bash
kubectl get clusterrole | grep rook
kubectl get clusterrolebinding | grep rook
kubectl get storageclass | grep rook
```

Expected: No output for each command.

### PersistentVolumes

```bash
kubectl get pv | grep -E "rook|ceph"
```

Expected: No output, or only `Released` PVs if you are intentionally keeping data.

### Secrets and ConfigMaps (cluster-scoped)

```bash
kubectl get secret -A | grep rook
kubectl get configmap -A | grep rook
```

Expected: No output.

## Layer 2: Node-Level State

SSH to each storage node and verify the following.

### Host Data Directory

```bash
for node in node-1 node-2 node-3 node-4 node-5 node-6; do
  echo "=== $node: dataDirHostPath ==="
  ssh $node "ls -la /var/lib/rook/ 2>/dev/null || echo 'Directory absent'"
done
```

Expected: Empty directory or directory absent on each node.

### Running Processes

```bash
for node in node-1 node-2 node-3 node-4 node-5 node-6; do
  echo "=== $node: Ceph processes ==="
  PROCS=$(ssh $node "ps aux | grep -E 'ceph-mon|ceph-osd|ceph-mgr|ceph-mds' | grep -v grep")
  if [ -n "$PROCS" ]; then
    echo "WARNING: Ceph processes still running on $node:"
    echo "$PROCS"
  else
    echo "OK: No Ceph processes"
  fi
done
```

Expected: "OK: No Ceph processes" on each node.

### Disk State

```bash
for node in node-1 node-2 node-3; do
  echo "=== $node: Disk labels ==="
  ssh $node "sudo wipefs /dev/sdb /dev/sdc /dev/sdd 2>/dev/null"
done
```

Expected: No output (no filesystem or Ceph labels on disks).

### LVM State

```bash
for node in node-1 node-2 node-3; do
  echo "=== $node: LVM ==="
  PVS=$(ssh $node "sudo pvs 2>/dev/null | grep ceph || true")
  VGS=$(ssh $node "sudo vgs 2>/dev/null | grep ceph || true")
  if [ -n "$PVS" ] || [ -n "$VGS" ]; then
    echo "WARNING: LVM Ceph volumes remain on $node"
    echo "$PVS"
    echo "$VGS"
  else
    echo "OK: No Ceph LVM volumes"
  fi
done
```

Expected: "OK: No Ceph LVM volumes" on each node.

### Kernel Modules

```bash
for node in node-1 node-2 node-3; do
  echo "=== $node: Kernel modules ==="
  MODS=$(ssh $node "lsmod | grep -E 'rbd|ceph' || true")
  if [ -n "$MODS" ]; then
    echo "WARNING: Ceph kernel modules still loaded on $node:"
    echo "$MODS"
  else
    echo "OK: No Ceph kernel modules"
  fi
done
```

## Layer 3: Automated Verification Script

Combine all checks into a single verification script:

```bash
#!/bin/bash

set -euo pipefail

NODES="${NODES:-node-1 node-2 node-3}"
PASS=0
FAIL=0

check() {
  local description="$1"
  local command="$2"
  local expected_empty="${3:-true}"

  RESULT=$(eval "$command" 2>/dev/null || true)
  if [ "$expected_empty" = "true" ] && [ -z "$RESULT" ]; then
    echo "PASS: $description"
    PASS=$((PASS + 1))
  elif [ "$expected_empty" = "true" ] && [ -n "$RESULT" ]; then
    echo "FAIL: $description"
    echo "  Found: $RESULT"
    FAIL=$((FAIL + 1))
  fi
}

echo "=== Kubernetes Layer ==="
check "rook-ceph namespace removed" "kubectl get namespace rook-ceph -o name"
check "Rook CRDs removed" "kubectl get crd -o name | grep -E 'rook|ceph'"
check "Rook ClusterRoles removed" "kubectl get clusterrole -o name | grep rook"
check "Rook StorageClasses removed" "kubectl get storageclass -o name | grep rook"
check "Rook PersistentVolumes removed" "kubectl get pv -o name | grep rook"

echo ""
echo "=== Node Layer ==="
for node in $NODES; do
  check "$node: dataDirHostPath empty" "ssh $node 'ls /var/lib/rook/ 2>/dev/null'"
  check "$node: No Ceph processes" "ssh $node 'ps aux | grep -E ceph-mon\|ceph-osd | grep -v grep'"
  check "$node: No Ceph kernel modules" "ssh $node 'lsmod | grep -E rbd\|ceph'"
done

echo ""
echo "=== Summary ==="
echo "PASS: $PASS"
echo "FAIL: $FAIL"

if [ "$FAIL" -eq 0 ]; then
  echo "Cleanup COMPLETE. Ready for fresh installation."
  exit 0
else
  echo "Cleanup INCOMPLETE. Address failures before reinstalling."
  exit 1
fi
```

## Summary

Verifying complete Rook-Ceph cleanup requires checking three layers: Kubernetes resources (namespace, CRDs, ClusterRoles, StorageClasses, PVs), node-level state (dataDirHostPath, Ceph processes, disk labels, LVM volumes, kernel modules), and confirming no orphaned secrets or configmaps remain. Use an automated verification script to check all layers systematically before declaring cleanup complete or attempting reinstallation. Any remaining artifacts will cause the new installation to conflict with the old cluster state.
