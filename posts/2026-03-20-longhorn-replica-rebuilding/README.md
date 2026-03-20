# How to Debug Longhorn Replica Rebuilding Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Replica Rebuilding, Kubernetes, Storage, Troubleshooting, Degraded Volume, SUSE Rancher

Description: Learn how to troubleshoot Longhorn replica rebuilding issues including stuck rebuilds, slow progress, and replica scheduling failures that leave volumes in a degraded state.

---

When a Longhorn replica fails or a new replica is added, Longhorn starts a rebuild process to synchronize data. If the rebuild gets stuck or fails repeatedly, volumes remain degraded and data redundancy is reduced.

---

## Step 1: Identify Replica Rebuild Status

```bash
# Check volume robustness - "degraded" means a replica is missing or rebuilding

kubectl get lhvolume -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,STATE:.status.state,ROBUSTNESS:.status.robustness,REPLICAS:.status.currentNodeID'

# Get detailed replica status for a specific volume
kubectl get lhreplica -n longhorn-system \
  -l longhornvolume=<volume-name> \
  -o wide
```

---

## Issue 1: Replica Stuck in "WaitForCleaning" State

This occurs when a failed replica process is not cleaning up properly.

```bash
# Check replica status
kubectl describe lhreplica <replica-name> -n longhorn-system

# Force delete the stuck replica
kubectl delete lhreplica <stuck-replica-name> -n longhorn-system --force

# Longhorn will schedule a new replica automatically
```

---

## Issue 2: Rebuild Not Starting Due to Scheduling Constraints

Longhorn cannot schedule a new replica if all suitable nodes are full or tainted.

```bash
# Check node scheduling status
kubectl get lhnode -n longhorn-system

# Check node conditions
kubectl get lhnode <node-name> -n longhorn-system -o yaml | grep -A 10 conditions

# If a node shows "Unschedulable: true", check its disk status
kubectl describe lhnode <node-name> -n longhorn-system
```

Fix: Free up disk space or add a new node with available storage.

---

## Issue 3: Slow Replica Rebuild

Long rebuild times cause volumes to remain degraded longer, increasing risk:

```bash
# Check rebuild progress via Longhorn UI (Volume > Replicas tab)
# Or check via the instance manager log:
kubectl logs -n longhorn-system \
  -l longhorn.io/component=instance-manager \
  --tail=200 | grep -i "rebuild\|sync"
```

To speed up rebuilds, increase rebuild concurrency in Longhorn settings:

```yaml
# Longhorn settings (via UI: Settings > Storage > Replica Rebuild Concurrent Limit)
concurrent-volume-backup-restore-per-node-limit: 2
```

---

## Issue 4: Replica Rebuild Fails with I/O Error

```bash
# Check dmesg for disk errors on the node
ssh <node> dmesg | grep -i "error\|I/O\|failed" | tail -20

# Check SMART status of the disk
smartctl -a /dev/sda

# If the disk is failing, evacuate volumes from that node
kubectl patch lhnode <node-name> -n longhorn-system \
  --type merge \
  -p '{"spec":{"allowScheduling":false,"evictionRequested":true}}'
```

---

## Issue 5: Too Many Concurrent Rebuilds Impacting Performance

```bash
# Check how many rebuilds are happening simultaneously
kubectl get lhreplica -n longhorn-system | grep RB | wc -l

# Limit concurrent rebuilds in Longhorn settings via UI:
# Settings > Storage > Concurrent Replica Rebuild Per Node Limit = 2
```

---

## Best Practices

- Set **replica count to 3** for production volumes - this gives you one failure tolerance while a rebuild completes.
- Monitor rebuild duration and alert if a rebuild takes longer than 2 hours.
- After adding new nodes, verify Longhorn can schedule replicas by running **Replica Auto Balance**.
- Keep node disk usage below 80% to ensure Longhorn always has space to schedule new replicas.
