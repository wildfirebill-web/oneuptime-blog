# How to Create Volumes in Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Storage, Longhorn, PVC

Description: Learn how to create and manage persistent storage volumes in Harvester for virtual machine disks and data storage.

## Introduction

Volumes in Harvester are backed by Longhorn, a distributed block storage system built into Kubernetes. Every VM disk is a Longhorn volume exposed as a Kubernetes PersistentVolumeClaim (PVC). You can create volumes independently and then attach them to VMs, or let Harvester create volumes automatically when provisioning a VM. This guide covers creating volumes manually for use cases like additional data disks.

## Volume Types in Harvester

| Volume Type | Description | Use Case |
|---|---|---|
| VM Root Disk | Created from a VM image, contains the OS | VM boot disk |
| Empty Data Volume | Blank volume, formatted by the VM | Additional storage for a VM |
| Imported Volume | Created from an existing image | Migrating VMs from other platforms |

## Step 1: Create a Volume via the UI

1. Log into the Harvester dashboard
2. Navigate to **Volumes** in the left sidebar
3. Click **Create**
4. Fill in the volume configuration:

```text
Name:           web-server-data
Namespace:      default
Size:           100 Gi
Storage Class:  longhorn  (default)
Access Mode:    ReadWriteOnce (for single VM attachment)
```

5. Click **Create** - the volume is created immediately

## Step 2: Create a Volume via kubectl

### Empty Data Volume

```yaml
# data-volume.yaml

# Create an empty 100 GB data volume for a VM

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-server-data
  namespace: default
  labels:
    # Mark as a Harvester-managed volume
    harvesterhci.io/managed: "true"
  annotations:
    harvesterhci.io/imageId: ""  # Empty for non-image volumes
spec:
  accessModes:
    - ReadWriteOnce  # Single VM attachment
  storageClassName: longhorn
  resources:
    requests:
      storage: 100Gi
```

```bash
kubectl apply -f data-volume.yaml

# Verify the volume was created and is in Available state
kubectl get pvc web-server-data -n default

# Expected output:
# NAME              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# web-server-data   Bound     pv-xxx   100Gi      RWO            longhorn       10s
```

### Volume from a VM Image

When creating a VM root disk, Harvester uses a DataVolume (CDI):

```yaml
# root-disk-from-image.yaml
# Create a 50 GB root disk from the Ubuntu 22.04 image

apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: ubuntu-vm-01-root
  namespace: default
  labels:
    harvesterhci.io/managed: "true"
  annotations:
    # Reference to the source VM image
    harvesterhci.io/imageId: "default/ubuntu-22-04-lts"
spec:
  source:
    pvc:
      # Source image PVC (created by Harvester from the VirtualMachineImage)
      namespace: default
      name: ubuntu-22-04-lts-image-pvc
  pvc:
    accessModes:
      - ReadWriteOnce
    storageClassName: longhorn
    resources:
      requests:
        storage: 50Gi
```

## Step 3: Configure Longhorn Replicas

Longhorn distributes replicas across nodes for data redundancy. Configure the default replica count:

```yaml
# longhorn-default-replicas.yaml
# Set default replica count to 3 for production data durability

apiVersion: longhorn.io/v1beta2
kind: Setting
metadata:
  name: default-replica-count
  namespace: longhorn-system
spec:
  value: "3"  # Replicate data across 3 nodes
```

```bash
kubectl apply -f longhorn-default-replicas.yaml

# Verify the setting
kubectl get setting default-replica-count -n longhorn-system \
    -o jsonpath='{.spec.value}'
```

To override the replica count for a specific volume:

```yaml
# high-replica-volume.yaml
# Volume with 3 replicas for critical data

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: critical-database-data
  namespace: default
  annotations:
    # Override default replicas for this volume
    longhorn.io/replica-count: "3"
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 500Gi
```

## Step 4: Create a StorageClass with Custom Settings

Create a custom StorageClass for different performance tiers:

```yaml
# storage-class-fast.yaml
# High-performance storage class for latency-sensitive workloads

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-fast
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  # Number of replicas
  numberOfReplicas: "2"
  # Node selector for replicas (requires nodes with SSD label)
  nodeSelector: "ssd=true"
  # Disk selector
  diskSelector: "nvme"
  # Data locality - prefer local replica for reads
  dataLocality: best-effort
  # Recurring job (hourly snapshot)
  recurringJobSelector: '[{"name":"snap","isGroup":false}]'
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

```bash
kubectl apply -f storage-class-fast.yaml

# Use the custom storage class in a volume
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-fast-volume
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-fast
  resources:
    requests:
      storage: 200Gi
EOF
```

## Step 5: Expand a Volume

Longhorn supports online volume expansion:

```bash
# Expand a volume from 100 GB to 200 GB
kubectl patch pvc web-server-data -n default \
    --type merge \
    -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# Watch the expansion progress
kubectl get pvc web-server-data -n default -w

# Inside the VM, you may need to resize the filesystem
# (for ext4)
sudo resize2fs /dev/vdb

# (for xfs)
sudo xfs_growfs /data
```

## Step 6: List and Monitor Volumes

```bash
# List all volumes in Harvester namespace
kubectl get pvc -n default

# List Longhorn volumes with replica details
kubectl get volumes.longhorn.io -n longhorn-system

# Get detailed info about a specific volume
kubectl describe volume.longhorn.io web-server-data -n longhorn-system

# Check replica health
kubectl get replicas.longhorn.io -n longhorn-system \
    -o custom-columns=\
'NAME:.metadata.name,VOLUME:.spec.volumeName,NODE:.spec.nodeID,STATE:.status.currentState'
```

## Step 7: Delete a Volume

```bash
# Delete a volume (must not be attached to a VM)
kubectl delete pvc web-server-data -n default

# Verify the Longhorn volume is also cleaned up
kubectl get volumes.longhorn.io -n longhorn-system | grep web-server-data
```

**Warning:** Deleting a PVC with `reclaimPolicy: Delete` permanently removes the data. Ensure you have a backup or snapshot before deleting.

## Conclusion

Volumes in Harvester are first-class Kubernetes resources powered by Longhorn's distributed storage. Creating volumes independently from VMs gives you flexibility to pre-provision storage, create data disks for existing VMs, and apply different storage policies to different workloads. Longhorn's replica mechanism ensures data durability across node failures, making it suitable for production VM storage. Always configure appropriate replica counts based on your availability requirements and node count.
