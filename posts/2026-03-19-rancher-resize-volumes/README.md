# How to Resize Persistent Volumes in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Storage, Persistent Volumes

Description: A practical guide to expanding persistent volume sizes in Rancher-managed Kubernetes clusters without data loss.

As applications grow, their storage needs increase. Kubernetes supports online and offline volume expansion, allowing you to resize persistent volumes without losing data. This guide explains how to resize PVs in Rancher-managed clusters.

## Prerequisites

- A running Rancher instance (v2.6 or later)
- A managed Kubernetes cluster
- A StorageClass with `allowVolumeExpansion: true`
- A CSI driver that supports volume expansion
- kubectl access to your cluster

## Step 1: Verify Volume Expansion Support

Check if your StorageClass allows expansion:

```bash
kubectl get storageclass -o custom-columns='NAME:.metadata.name,EXPANSION:.allowVolumeExpansion'
```

If `allowVolumeExpansion` is not `true`, update the StorageClass:

```bash
kubectl patch storageclass <class-name> -p '{"allowVolumeExpansion": true}'
```

Check if your CSI driver supports expansion:

```bash
kubectl get csidrivers -o custom-columns='NAME:.metadata.name,EXPAND:.spec.requiresRepublish'
```

## Step 2: Check Current Volume Size

```bash
kubectl get pvc my-pvc -n default -o custom-columns='NAME:.metadata.name,SIZE:.spec.resources.requests.storage,STATUS:.status.phase'

kubectl describe pvc my-pvc -n default | grep -A 2 Capacity
```

## Step 3: Resize the PVC

Edit the PVC to request more storage:

```bash
kubectl patch pvc my-pvc -n default -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
```

Or edit the YAML directly:

```bash
kubectl edit pvc my-pvc -n default
```

Change the `storage` field under `spec.resources.requests`:

```yaml
spec:
  resources:
    requests:
      storage: 50Gi  # changed from 20Gi
```

## Step 4: Resize via the Rancher UI

1. Navigate to your cluster in Rancher.
2. Go to **Storage** > **PersistentVolumeClaims**.
3. Find the PVC you want to resize.
4. Click the three-dot menu and select **Edit Config**.
5. Update the **Capacity** field to the new size.
6. Click **Save**.

## Step 5: Monitor the Resize Operation

Check the resize progress:

```bash
kubectl get pvc my-pvc -n default -o yaml | grep -A 5 conditions
```

During resize, you will see a condition like:

```yaml
conditions:
- type: FileSystemResizePending
  status: "True"
  message: "Waiting for user to (re-)start a pod to finish file system resize"
```

After the filesystem is resized:

```yaml
conditions:
- type: Resizing
  status: "False"
```

## Step 6: Handle FileSystem Resize

Some CSI drivers require a pod restart to complete the filesystem resize:

```bash
# Check if filesystem resize is pending
kubectl get pvc my-pvc -n default -o jsonpath='{.status.conditions[*].type}'
```

If you see `FileSystemResizePending`, restart the pod:

```bash
# For a Deployment
kubectl rollout restart deployment my-app -n default

# For a StatefulSet
kubectl delete pod my-statefulset-0 -n default
# The StatefulSet controller will recreate it
```

## Step 7: Verify the New Size

After the resize is complete:

```bash
# Check PVC size
kubectl get pvc my-pvc -n default

# Check PV size
kubectl get pv $(kubectl get pvc my-pvc -n default -o jsonpath='{.spec.volumeName}')

# Verify inside the pod
kubectl exec <pod-name> -n default -- df -h /data
```

## Step 8: Resize Volumes in StatefulSets

For StatefulSets, resize each PVC individually:

```bash
# List PVCs for the StatefulSet
kubectl get pvc -n default -l app=database

# Resize each PVC
for i in 0 1 2; do
  kubectl patch pvc data-database-$i -n default \
    -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
done
```

Then restart the pods to complete filesystem resize:

```bash
for i in 0 1 2; do
  kubectl delete pod database-$i -n default
  # Wait for pod to be ready before deleting the next
  kubectl wait --for=condition=ready pod/database-$i -n default --timeout=120s
done
```

## Step 9: Automate Volume Resizing

Create a script to monitor and resize volumes when they reach a threshold:

```bash
#!/bin/bash
THRESHOLD=80
NAMESPACE="default"

for pvc in $(kubectl get pvc -n $NAMESPACE -o jsonpath='{.items[*].metadata.name}'); do
  POD=$(kubectl get pods -n $NAMESPACE -o jsonpath="{.items[?(@.spec.volumes[*].persistentVolumeClaim.claimName=='$pvc')].metadata.name}" | head -1)

  if [ -z "$POD" ]; then
    continue
  fi

  MOUNT=$(kubectl get pvc $pvc -n $NAMESPACE -o jsonpath='{.spec.volumeName}')
  USAGE=$(kubectl exec $POD -n $NAMESPACE -- df --output=pcent /data 2>/dev/null | tail -1 | tr -d ' %')

  if [ ! -z "$USAGE" ] && [ "$USAGE" -gt "$THRESHOLD" ]; then
    CURRENT=$(kubectl get pvc $pvc -n $NAMESPACE -o jsonpath='{.spec.resources.requests.storage}')
    echo "PVC $pvc usage at ${USAGE}%, current size: $CURRENT"
  fi
done
```

## Step 10: Handle Resize Failures

If a resize operation fails:

```bash
# Check PVC events
kubectl describe pvc my-pvc -n default

# Check CSI controller logs
kubectl logs -n kube-system -l app=csi-controller --tail=100 | grep -i resize

# Check conditions
kubectl get pvc my-pvc -n default -o jsonpath='{.status.conditions}' | jq .
```

Common failure reasons:
- StorageClass does not allow expansion
- CSI driver does not support expansion
- Backend storage has insufficient capacity
- Volume is at maximum size for its type

## Important Notes

- Volume expansion is a one-way operation; you cannot shrink volumes
- Always ensure you have backups before resizing
- Some storage backends require the volume to be detached during resize
- Online expansion support varies by CSI driver
- The PV reclaim policy does not affect resize operations

## Summary

Resizing persistent volumes in Rancher is a straightforward process when your StorageClass and CSI driver support volume expansion. The key steps are patching the PVC with a larger storage request and, if needed, restarting pods to complete the filesystem resize. Always verify expansion support before attempting to resize, and maintain backups as a safety measure.
