# How to Troubleshoot Longhorn Volume Attachment Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Troubleshooting, Volume Attachment, Kubernetes, Storage, Debugging, SUSE Rancher

Description: Learn how to diagnose and fix Longhorn volume attachment failures including volumes stuck in attaching state, node-to-replica communication errors, and iSCSI connectivity issues.

---

Volume attachment failures are one of the most common Longhorn operational issues. A volume stuck in `Attaching` state prevents pods from starting and blocks deployments. This guide covers the main causes and fixes.

---

## Step 1: Identify the Problem

```bash
# Check which volumes are stuck
kubectl get lhvolume -n longhorn-system | grep -v healthy

# Get detailed status of a specific volume
kubectl describe lhvolume <volume-name> -n longhorn-system

# Check the PVC binding status
kubectl get pvc -A | grep -v Bound
```

---

## Common Causes and Fixes

### Cause 1: All Replicas Are on Unavailable Nodes

```bash
# Check replica status
kubectl get lhreplica -n longhorn-system \
  -l longhornvolume=<volume-name>

# If replicas are on nodes that are NotReady
kubectl get nodes

# Option 1: Bring the node back online
# Option 2: Wait for the node to be detected as offline and replicas to failover
# Force detach and re-attach to trigger replica rebuild:
kubectl patch lhvolume <volume-name> -n longhorn-system \
  --type merge \
  -p '{"spec":{"nodeID":""}}'
```

---

### Cause 2: iSCSI/NVMe Initiator Not Installed on Node

```bash
# Check if open-iscsi is installed
iscsiadm --version

# Install if missing
sudo apt-get install -y open-iscsi
sudo systemctl enable --now iscsid

# For NVMe-oF based volumes (Longhorn v1.5+)
sudo modprobe nvme-tcp
```

---

### Cause 3: Volume Is Attached to a Different Node

Longhorn RWO volumes can only be attached to one node at a time. If the node is gone but the attachment record remains:

```bash
# List all volume attachments
kubectl get volumeattachment

# Delete the stale attachment
kubectl delete volumeattachment <attachment-name>
```

---

### Cause 4: Engine Process Crashed

```bash
# Check Longhorn instance manager pods
kubectl get pods -n longhorn-system -l longhorn.io/component=instance-manager

# Check engine manager logs on the node where the volume is hosted
kubectl logs -n longhorn-system \
  -l longhorn.io/component=instance-manager \
  --tail=100

# Restart the instance manager pod (it will respawn automatically)
kubectl delete pod -n longhorn-system \
  -l longhorn.io/component=instance-manager \
  --field-selector spec.nodeName=<node-name>
```

---

### Cause 5: Insufficient Disk Space

```bash
# Check node disk usage
kubectl get lhnode -n longhorn-system -o wide

# Check Longhorn node storage
kubectl get lhnode <node-name> -n longhorn-system \
  -o jsonpath='{.status.diskStatus}'
```

---

## Step 5: Force Volume Detach (Last Resort)

If a volume is stuck and preventing pod scheduling:

```bash
# Force detach via Longhorn UI: Volume > Detach (with force)
# Or via API:
LONGHORN_URL=http://longhorn-frontend.longhorn-system.svc.cluster.local
curl -X POST \
  "${LONGHORN_URL}/v1/volumes/<volume-name>?action=detach" \
  -H "Content-Type: application/json" \
  -d '{"hostId":""}'
```

---

## Best Practices

- Install `open-iscsi` on all nodes **before** installing Longhorn — it is a hard requirement.
- Use `PodDisruptionBudgets` to ensure pods using Longhorn volumes are drained safely during node maintenance.
- Enable **Longhorn's automatic node draining** via the Longhorn settings to handle graceful pod migrations.
