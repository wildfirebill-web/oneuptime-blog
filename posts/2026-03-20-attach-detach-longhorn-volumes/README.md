# How to Attach and Detach Longhorn Volumes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, Volumes, Operations

Description: Learn how to attach and detach Longhorn volumes from nodes using the Longhorn UI, kubectl, and the Longhorn API for maintenance and debugging scenarios.

## Introduction

Longhorn volumes are normally attached and detached automatically as pods that consume them are scheduled and terminated. However, there are scenarios - such as data migration, debugging, or maintenance - where you need to manually attach or detach a volume from a node. This guide covers both the automatic lifecycle and the manual attach/detach process.

## Understanding Volume Attachment

In Longhorn, a volume can be in one of the following states:
- **Detached**: The volume exists but is not mounted to any node
- **Attaching**: The volume is in the process of being attached
- **Attached**: The volume is attached to a node and accessible
- **Detaching**: The volume is being detached

## Automatic Attachment via Kubernetes

The most common way volumes are attached is automatically through Kubernetes. When a pod using a PVC is scheduled on a node, Longhorn attaches the volume to that node automatically.

```yaml
# pod-with-pvc.yaml - Kubernetes automatically attaches the volume when this pod runs

apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-app-data   # PVC backed by Longhorn
```

```bash
kubectl apply -f pod-with-pvc.yaml

# Verify the volume is attached
kubectl get volumeattachments
```

## Manual Attachment via Longhorn UI

### Attach a Volume

1. Open the Longhorn UI
2. Navigate to **Volume**
3. Find the volume you want to attach (it should be in **Detached** state)
4. Click the three-dot menu (⋮) next to the volume
5. Select **Attach**
6. Choose the target node from the dropdown
7. Check **Maintenance Mode** if you want to attach without making it available to pods
8. Click **OK**

### Detach a Volume

1. In the Longhorn UI, navigate to **Volume**
2. Find the attached volume
3. Ensure no pod is currently using the volume
4. Click the three-dot menu (⋮) next to the volume
5. Select **Detach**
6. Confirm the operation

## Manual Attachment via kubectl

Longhorn uses a `VolumeAttachment` custom resource for managing attachment state:

```yaml
# longhorn-volume-attachment.yaml - Manually attach a Longhorn volume to a node
apiVersion: longhorn.io/v1beta2
kind: VolumeAttachment
metadata:
  name: va-my-volume
  namespace: longhorn-system
spec:
  volume: my-longhorn-volume   # Name of the Longhorn volume
  attachmentTickets:
    manual-ticket:
      id: manual-ticket
      type: longhorn-api
      parameters:
        # Specify the node to attach the volume to
        nodeID: "worker-node-1"
        disableFrontend: "false"
```

```bash
kubectl apply -f longhorn-volume-attachment.yaml

# Check attachment status
kubectl get volumeattachments.longhorn.io -n longhorn-system
```

## Manual Detachment via kubectl

To detach a volume that was manually attached:

```bash
# Delete the VolumeAttachment resource to trigger detachment
kubectl delete volumeattachment.longhorn.io va-my-volume -n longhorn-system
```

## Attaching a Volume for Data Access Without a Pod

Sometimes you need to access a volume's data for debugging or migration without running a workload. You can attach it to a node and mount it manually:

```bash
# Step 1: Get the Longhorn volume name
kubectl get volumes.longhorn.io -n longhorn-system

# Step 2: Patch the volume to trigger attachment to a specific node
# Using the Longhorn API via kubectl
kubectl patch volume.longhorn.io my-volume -n longhorn-system \
  --type merge \
  -p '{"spec": {"nodeID": "worker-node-1"}}'

# Step 3: Wait for the volume to attach
kubectl get volumes.longhorn.io my-volume -n longhorn-system -w

# Step 4: SSH to the node and find the device
ls /dev/longhorn/
# The volume will appear as /dev/longhorn/my-volume

# Step 5: Mount the device
mount /dev/longhorn/my-volume /mnt/data

# Step 6: Perform your data access

# Step 7: Unmount before detaching
umount /mnt/data
```

## Checking Volume Attachment Status

```bash
# Check Kubernetes VolumeAttachment objects
kubectl get volumeattachments

# Check Longhorn volume status
kubectl get volumes.longhorn.io -n longhorn-system

# Get detailed volume information including attachment state
kubectl describe volume.longhorn.io <volume-name> -n longhorn-system | grep -A 10 "Status:"
```

## Handling Stuck Attachments

If a volume is stuck in an attaching or detaching state:

```bash
# Force delete the VolumeAttachment if it is stuck
kubectl get volumeattachments | grep <volume-name>
kubectl delete volumeattachment <attachment-name>

# Restart the Longhorn manager on the affected node
kubectl delete pod -n longhorn-system \
  -l app=longhorn-manager \
  --field-selector spec.nodeName=<stuck-node>
```

## Conclusion

Longhorn's volume attachment system works seamlessly with Kubernetes to automatically manage volume lifecycle. For maintenance and debugging scenarios, both the UI and kubectl provide straightforward mechanisms to manually control attachment state. Understanding how to work with volume attachments is essential for troubleshooting storage issues and performing data migrations in a Longhorn-managed cluster.
