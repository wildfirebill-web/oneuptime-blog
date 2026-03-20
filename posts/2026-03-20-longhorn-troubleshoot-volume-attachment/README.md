# How to Troubleshoot Longhorn Volume Attachment Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Troubleshooting, Volume Attachment, Kubernetes, Storage, Debugging, SUSE Rancher

Description: Learn how to diagnose and fix Longhorn volume attachment failures including stuck volumes, CSI driver issues, node selector problems, and iSCSI connectivity errors.

---

Volume attachment failures prevent pods from starting and can cause application outages. Longhorn volume attachment issues typically relate to CSI driver health, node availability, or network connectivity to storage replicas.

---

## Common Volume Attachment Symptoms

| Symptom | Likely Cause |
|---|---|
| Pod stuck in `ContainerCreating` | Volume not attached to the node |
| PVC in `Pending` state | No matching node for volume |
| Volume in `Attaching` state (long time) | Node or replica unreachable |
| `Multi-Attach error` | Volume attached to wrong node |

---

## Step 1: Check the Pod Event Log

```bash
# Describe the pod to see attachment errors

kubectl describe pod <pod-name> -n <namespace>

# Look for events like:
# Warning  FailedMount    <time>   kubelet   Unable to attach or mount volumes
# Warning  FailedAttachVolume  <time>  attachdetach  AttachVolume.Attach failed
```

---

## Step 2: Check the Longhorn Volume Status

```bash
# Get the volume associated with the PVC
PVC_NAME=my-pvc
NAMESPACE=default
PV_NAME=$(kubectl get pvc $PVC_NAME -n $NAMESPACE -o jsonpath='{.spec.volumeName}')
VOLUME_NAME=$(kubectl get pv $PV_NAME -o jsonpath='{.spec.csi.volumeHandle}')

# Check the Longhorn volume status
kubectl get volume -n longhorn-system $VOLUME_NAME -o yaml

# Key fields to check:
# status.state: should be "attached" when mounted
# status.robustness: should be "healthy"
# status.conditions: look for error conditions
```

---

## Step 3: Check the VolumeAttachment Object

```bash
# List Kubernetes VolumeAttachment objects
kubectl get volumeattachment

# Describe the attachment
kubectl describe volumeattachment <attachment-name>

# If attachment is stuck, delete and let it recreate
kubectl delete volumeattachment <attachment-name>
```

---

## Step 4: Check Longhorn CSI Driver Pods

```bash
# Check CSI driver pods are healthy
kubectl get pods -n longhorn-system | grep csi

# Expected pods:
# longhorn-csi-attacher       (handles volume attachment)
# longhorn-csi-provisioner    (handles PVC provisioning)
# longhorn-csi-resizer        (handles volume resize)
# longhorn-csi-snapshotter    (handles snapshots)

# Check CSI attacher logs for errors
kubectl logs -n longhorn-system \
  $(kubectl get pod -n longhorn-system -l app=csi-attacher -o name | head -1) \
  -c csi-attacher
```

---

## Step 5: Check the Longhorn Manager and Instance Manager

```bash
# Check Longhorn manager on the target node
NODE=<node-name>
kubectl get pod -n longhorn-system \
  -l app=longhorn-manager \
  --field-selector spec.nodeName=$NODE

# View manager logs
kubectl logs -n longhorn-system \
  $(kubectl get pod -n longhorn-system -l app=longhorn-manager \
    --field-selector spec.nodeName=$NODE -o name) \
  | grep -i "error\|attach\|volume"

# Check instance manager
kubectl get pod -n longhorn-system \
  -l longhorn.io/component=instance-manager,longhorn.io/node=$NODE
```

---

## Step 6: Fix a Stuck Volume Detach

If a volume is stuck in `Detaching` state (usually after a node crash):

```bash
# In the Longhorn UI: Volumes → select volume → Detach

# Or via kubectl - patch the volume to force detach
kubectl patch volume $VOLUME_NAME \
  -n longhorn-system \
  --type merge \
  -p '{"spec":{"nodeID":""}}'
```

---

## Step 7: Check iSCSI Connectivity

Longhorn uses iSCSI for volume I/O. If iSCSI is blocked, volumes won't attach:

```bash
# Check if iscsid is running on the node
kubectl debug node/<node-name> -it --image=alpine -- \
  sh -c "apk add iproute2 && ss -tlnp | grep 3260"

# Check if iscsi_tcp module is loaded
kubectl debug node/<node-name> -it --image=alpine -- \
  sh -c "cat /proc/modules | grep iscsi"

# Load the module if missing
kubectl debug node/<node-name> -it --image=alpine -- \
  modprobe iscsi_tcp
```

---

## Step 8: Check Node Schedulability for Longhorn

```bash
# Check if the node is allowed to schedule Longhorn volumes
kubectl get node <node-name> -o yaml | grep -A 10 taints

# Check if Longhorn node selector matches
kubectl get setting -n longhorn-system \
  default-longhorn-static-storage-class -o yaml

# View node disk status in Longhorn
kubectl get node.longhorn.io -n longhorn-system
```

---

## Best Practices

- Check the Longhorn dashboard first - it shows volume state, replica health, and which node a volume is attached to at a glance.
- If a node crashes with volumes attached, wait for the node to come back up before attempting manual detachment - Longhorn will auto-reattach when the node rejoins.
- Ensure `iscsid` is running on all nodes and the `iscsi_tcp` kernel module is loaded - these are the most common causes of attachment failures on fresh nodes.
