# How to Troubleshoot Longhorn Node Not Schedulable

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, Troubleshooting, Nodes, Scheduling

Description: Diagnose and fix issues that cause Longhorn nodes to be marked as not schedulable, preventing new volumes from being created or replicas from being placed.

## Introduction

When a Longhorn node is not schedulable, it cannot accept new volume replicas. This can prevent the creation of new PVCs, block replica rebuilding, and leave volumes in a degraded state with fewer replicas than desired. Understanding why a node becomes non-schedulable is key to restoring full cluster storage capacity.

## Understanding Schedulability

A Longhorn node is considered schedulable when:
1. The node itself allows scheduling (`spec.allowScheduling: true`)
2. At least one disk on the node has available space above the minimum threshold
3. The Kubernetes node is Ready and not cordoned
4. The node has no conditions that prevent scheduling

## Initial Diagnosis

```bash
# Check which Longhorn nodes are not schedulable
kubectl get nodes.longhorn.io -n longhorn-system \
  -o custom-columns="NAME:.metadata.name,SCHEDULABLE:.spec.allowScheduling,STATUS:.status.conditions"

# Check the conditions on all nodes
kubectl describe nodes.longhorn.io -n longhorn-system | \
  grep -A 10 "Conditions:"

# Look for nodes with "Schedulable: false"
kubectl get nodes.longhorn.io -n longhorn-system -o yaml | \
  grep -B 5 "allowScheduling: false"
```

## Issue 1: Manual Scheduling Disabled

The most straightforward cause — scheduling was explicitly disabled.

```bash
# Check if scheduling is disabled
kubectl get nodes.longhorn.io <node-name> -n longhorn-system \
  -o jsonpath='{.spec.allowScheduling}'

# Re-enable scheduling
kubectl patch nodes.longhorn.io <node-name> \
  -n longhorn-system \
  --type merge \
  -p '{"spec": {"allowScheduling": true}}'
```

## Issue 2: Disk Space Below Minimum Threshold

```bash
# Check available storage on all nodes
kubectl get nodes.longhorn.io -n longhorn-system -o yaml | \
  grep -A 10 "diskStatus:"

# Check the minimum available percentage setting
kubectl get settings.longhorn.io storage-minimal-available-percentage \
  -n longhorn-system -o yaml
# Default is 25 — if available storage < 25% of total, node becomes non-schedulable

# Check actual disk usage
kubectl get nodes.longhorn.io <node-name> -n longhorn-system -o yaml | \
  jq '.status.diskStatus | to_entries[] | {disk: .key, available: .value.storageAvailable, max: .value.storageMaximum}'
```

### Solution: Free Up Disk Space

```bash
# 1. Delete unused PVCs
kubectl get pvc --all-namespaces | grep -v Bound
kubectl delete pvc <unused-pvc> -n <namespace>

# 2. Clean up old Longhorn replicas (orphaned replicas)
kubectl get replicas.longhorn.io -n longhorn-system | grep ERR

# 3. Reduce the minimum available percentage temporarily
kubectl patch settings.longhorn.io storage-minimal-available-percentage \
  -n longhorn-system \
  --type merge \
  -p '{"value": "10"}'  # Reduce temporarily

# 4. Or increase over-provisioning
kubectl patch settings.longhorn.io storage-over-provisioning-percentage \
  -n longhorn-system \
  --type merge \
  -p '{"value": "300"}'
```

## Issue 3: Kubernetes Node is Cordoned

If the Kubernetes node itself is cordoned, Longhorn also marks it as non-schedulable.

```bash
# Check Kubernetes node status
kubectl get nodes
# Look for "SchedulingDisabled" in STATUS column

# Check if node is cordoned
kubectl get node <node-name> -o yaml | grep unschedulable

# Uncordon the node
kubectl uncordon <node-name>

# Verify Longhorn reflects the change (may take a moment)
kubectl get nodes.longhorn.io <node-name> -n longhorn-system
```

## Issue 4: Node is Not Ready in Kubernetes

```bash
# Check node Ready condition
kubectl describe node <node-name> | grep -A 5 "Conditions:"

# Look for specific issues:
# - NotReady: node is unreachable
# - DiskPressure: disk is almost full
# - MemoryPressure: node is low on memory
# - PIDPressure: too many processes
```

### Fix DiskPressure

```bash
# SSH to the node and check disk usage
df -h

# Clean up container images
crictl rmi --prune

# Clean up old log files
journalctl --vacuum-size=500M

# Check for large files
du -sh /var/lib/longhorn/*
du -sh /var/log/*
```

## Issue 5: Longhorn Manager Pod Not Running on the Node

```bash
# Check if the Longhorn manager DaemonSet pod is running on the node
kubectl get pods -n longhorn-system \
  --field-selector spec.nodeName=<node-name> \
  -l app=longhorn-manager

# If the pod is not running, check for errors
kubectl describe pod -n longhorn-system \
  -l app=longhorn-manager \
  --field-selector spec.nodeName=<node-name>

# Restart the manager pod
kubectl delete pod -n longhorn-system \
  -l app=longhorn-manager \
  --field-selector spec.nodeName=<node-name>
```

## Issue 6: Disk Not Detected by Longhorn

```bash
# Check disks configured for the node
kubectl get nodes.longhorn.io <node-name> \
  -n longhorn-system -o yaml | grep -A 20 "spec:"

# Verify the disk path exists on the node
# SSH to the node and check
ls -la /var/lib/longhorn/

# If a disk was removed, remove it from Longhorn config
# Via Longhorn UI: Node → Edit → Remove disk
```

## Automated Schedulability Checking

Set up a script to check and alert on non-schedulable nodes:

```bash
# check-longhorn-nodes.sh - Alert on non-schedulable nodes
#!/bin/bash

NON_SCHEDULABLE=$(kubectl get nodes.longhorn.io -n longhorn-system \
  -o json | \
  jq -r '.items[] |
    select(
      .spec.allowScheduling == false or
      (.status.conditions[] | select(.type == "Schedulable") | .status) == "False"
    ) |
    .metadata.name')

if [ -n "$NON_SCHEDULABLE" ]; then
  echo "ALERT: The following Longhorn nodes are not schedulable:"
  echo "$NON_SCHEDULABLE"
  exit 1
else
  echo "All Longhorn nodes are schedulable"
  exit 0
fi
```

## Checking Node Conditions

Longhorn reports conditions on each node. Check these for detailed status:

```bash
# Get all node conditions
kubectl get nodes.longhorn.io -n longhorn-system \
  -o json | \
  jq -r '.items[] |
    .metadata.name as $name |
    .status.conditions[] |
    "\($name): \(.type) = \(.status) (\(.message))"'
```

Key conditions to look for:
- `Schedulable`: Whether the node can accept new replicas
- `Ready`: Whether the node is connected to Longhorn
- `MountPropagation`: Whether mount propagation is configured correctly

## Conclusion

Longhorn nodes becoming non-schedulable is a common issue that usually stems from disk space exhaustion, manual scheduling disabling, or node-level problems in Kubernetes. By regularly monitoring node schedulability through Prometheus alerts and the Longhorn UI, you can catch and address these issues before they impact volume health. Maintaining adequate disk headroom — at least 30% free space — is the most effective preventive measure.
