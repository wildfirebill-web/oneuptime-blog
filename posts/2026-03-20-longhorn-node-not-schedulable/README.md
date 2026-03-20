# How to Fix Longhorn Node Not Schedulable Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Troubleshooting, Node Scheduling, Kubernetes, Storage, SUSE Rancher

Description: Learn how to diagnose and fix Longhorn nodes marked as not schedulable for replica placement, covering disk pressure, condition checks, and allowScheduling configuration.

---

When a Longhorn node is marked as "not schedulable", Longhorn will not place new replicas on it. This reduces available storage capacity and can cause volumes to remain degraded if there are not enough schedulable nodes.

---

## Step 1: Identify Non-Schedulable Nodes

```bash
# List Longhorn nodes and their scheduling status

kubectl get lhnode -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,SCHEDULABLE:.status.conditions[?(@.type=="Schedulable")].status,READY:.status.conditions[?(@.type=="Ready")].status'

# Get detailed status of a specific Longhorn node
kubectl describe lhnode <node-name> -n longhorn-system
```

---

## Cause 1: Disk Space Exceeded Threshold

Longhorn stops scheduling replicas on a node when disk usage exceeds the threshold (default 85%).

```bash
# Check disk usage on the Longhorn node
kubectl get lhnode <node-name> -n longhorn-system \
  -o jsonpath='{.status.diskStatus}' | jq .

# Check actual disk usage on the host
df -h /var/lib/longhorn
```

Fix: Free disk space or add a new disk to the node.

---

## Cause 2: Node Scheduling Disabled Manually

```bash
# Check if scheduling was manually disabled
kubectl get lhnode <node-name> -n longhorn-system \
  -o jsonpath='{.spec.allowScheduling}'

# Re-enable scheduling
kubectl patch lhnode <node-name> -n longhorn-system \
  --type merge \
  -p '{"spec":{"allowScheduling":true}}'
```

---

## Cause 3: Longhorn Disk Not Configured Properly

```bash
# List disks configured on the node
kubectl get lhnode <node-name> -n longhorn-system \
  -o jsonpath='{.spec.disks}' | jq .

# Check disk conditions
kubectl get lhnode <node-name> -n longhorn-system \
  -o yaml | grep -A 20 diskStatus
```

If a disk shows `diskReady: false`, check:

```bash
# On the host node - check if the disk path exists
ls -la /var/lib/longhorn

# Check if Longhorn can write to the disk
touch /var/lib/longhorn/test && rm /var/lib/longhorn/test
```

---

## Cause 4: Node Kubernetes Taint Prevents Longhorn Scheduling

```bash
# Check node taints
kubectl describe node <node-name> | grep Taints

# If the node has a custom taint, add the taint to Longhorn's global settings
# Settings > Taint Toleration in Longhorn UI
```

Add toleration in Longhorn settings:

```yaml
# In Longhorn UI: Settings > Taint Toleration
# Format: key=value:effect
dedicated=storage:NoSchedule
```

---

## Cause 5: Node Eviction Was Requested

```bash
# Check if eviction is requested
kubectl get lhnode <node-name> -n longhorn-system \
  -o jsonpath='{.spec.evictionRequested}'

# Cancel eviction
kubectl patch lhnode <node-name> -n longhorn-system \
  --type merge \
  -p '{"spec":{"evictionRequested":false}}'
```

---

## Best Practices

- Keep Longhorn disk usage below 80% - the 85% threshold doesn't leave enough buffer for replica rebuilds.
- Use separate dedicated disks for Longhorn storage on each node for better isolation.
- Regularly check `kubectl get lhnode` in your monitoring dashboard to catch scheduling issues before they impact data availability.
