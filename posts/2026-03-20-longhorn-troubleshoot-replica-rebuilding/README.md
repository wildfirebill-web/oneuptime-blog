# How to Troubleshoot Longhorn Replica Rebuilding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Troubleshooting, Replica Rebuilding, Storage, Kubernetes, Data Recovery, SUSE Rancher

Description: Learn how to diagnose and resolve Longhorn replica rebuilding failures, understand replica states, manage rebuilding priorities, and recover degraded volumes.

---

Longhorn volumes become degraded when a replica fails or a node goes offline. The replica rebuilding process restores redundancy by copying data to a new replica. Rebuilding failures can leave volumes in a degraded state, increasing the risk of data loss.

---

## Understanding Replica States

| State | Description |
|---|---|
| Running | Replica is healthy and in sync |
| Rebuilding | Replica is currently receiving data |
| Failed | Replica has failed, needs to be replaced |
| Stopped | Replica is stopped (node offline) |
| Deleted | Replica has been removed |

---

## Step 1: Check Volume and Replica Status

```bash
# List all volumes and their robustness
kubectl get volume -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,STATE:.status.state,ROBUSTNESS:.status.robustness'

# Check replicas for a specific volume
kubectl get replica -n longhorn-system \
  -l longhornvolume=<volume-name> \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeID,STATE:.status.currentState'

# Describe a specific replica for error details
kubectl describe replica <replica-name> -n longhorn-system
```

---

## Step 2: Check Rebuilding Progress

```bash
# View rebuilding progress via the Longhorn API
kubectl exec -n longhorn-system \
  $(kubectl get pod -n longhorn-system -l app=longhorn-manager -o name | head -1) \
  -- curl -s http://localhost:9500/v1/volumes/<volume-name> \
  | jq '.rebuildStatus'

# Or use the Longhorn UI:
# Volumes → select volume → Replicas tab
# Look for "Rebuilding: X%" progress
```

---

## Step 3: Diagnose Rebuilding Failures

```bash
# Check Longhorn manager logs for rebuilding errors
kubectl logs -n longhorn-system \
  $(kubectl get pod -n longhorn-system -l app=longhorn-manager -o name | head -1) \
  | grep -i "rebuild\|replica\|error" | tail -50

# Check instance manager logs on the node running the replica
kubectl logs -n longhorn-system \
  $(kubectl get pod -n longhorn-system \
    -l longhorn.io/component=instance-manager \
    --field-selector spec.nodeName=<target-node> -o name | head -1)
```

---

## Common Issue 1: Rebuilding Stalls Due to Disk Space

```bash
# Check disk usage on all nodes
kubectl get node.longhorn.io -n longhorn-system -o yaml \
  | grep -A 5 "diskStatus"

# Check disk space directly on the node
kubectl debug node/<node-name> -it --image=alpine -- \
  df -h /var/lib/longhorn

# If disk is full, expand the disk or add a new disk to Longhorn
# Longhorn UI → Node → Edit → Add Disk
```

---

## Common Issue 2: Rebuilding Fails Due to Network Issues

```bash
# Check network connectivity between nodes
# Longhorn replicas communicate on port 8503

kubectl debug node/<source-node> -it --image=alpine -- \
  nc -zv <replica-node-ip> 8503

# Check for NetworkPolicy blocking Longhorn traffic
kubectl get networkpolicy -A | grep longhorn
```

---

## Common Issue 3: Manual Replica Removal and Rebuild

If a replica is stuck in a failed state and not rebuilding automatically:

```bash
# Delete the failed replica (Longhorn will create a new one)
kubectl delete replica <failed-replica-name> -n longhorn-system

# Monitor the new replica being created
kubectl get replica -n longhorn-system \
  -l longhornvolume=<volume-name> -w
```

---

## Step 4: Configure Rebuilding Settings

```bash
# Adjust concurrent rebuild limit (default: 5)
kubectl patch setting -n longhorn-system \
  concurrent-replica-rebuild-per-node-limit \
  --type merge \
  -p '{"value":"2"}'

# Adjust replica rebuild timeout (seconds, default: 600)
kubectl patch setting -n longhorn-system \
  replica-replenishment-wait-interval \
  --type merge \
  -p '{"value":"600"}'
```

---

## Step 5: Force a Volume to Rebuild

```bash
# Detach and reattach the volume to trigger rebuilding
# First, scale down the workload
kubectl scale deployment <deployment-name> --replicas=0

# Wait for the volume to detach
kubectl get volume -n longhorn-system <volume-name> -w

# Scale back up — Longhorn will reattach and trigger rebuild
kubectl scale deployment <deployment-name> --replicas=1
```

---

## Step 6: Verify Rebuild Completion

```bash
# Volume should show "healthy" after rebuild completes
kubectl get volume -n longhorn-system <volume-name> \
  -o jsonpath='{.status.robustness}'

# All replicas should be "running"
kubectl get replica -n longhorn-system \
  -l longhornvolume=<volume-name> \
  -o jsonpath='{.items[*].status.currentState}'
```

---

## Best Practices

- Monitor Longhorn volume robustness with Prometheus alerts — a volume in `degraded` state means rebuilding is needed and the risk of data loss is elevated.
- Set `concurrent-replica-rebuild-per-node-limit` to a value that won't overwhelm node I/O during peak hours — rebuilding is I/O intensive.
- Keep at least 30% free disk space on Longhorn nodes — rebuilding requires temporary extra space on the target node.
