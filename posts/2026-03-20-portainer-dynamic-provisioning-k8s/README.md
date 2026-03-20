# How to Configure Dynamic Provisioning for Kubernetes Storage in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Storage, StorageClass, DevOps

Description: Learn how to configure dynamic volume provisioning for Kubernetes storage in Portainer using StorageClasses and PersistentVolumeClaims.

## Introduction

Dynamic provisioning in Kubernetes allows PersistentVolumes (PVs) to be created automatically when a PersistentVolumeClaim (PVC) is submitted. Portainer provides a UI-friendly way to manage StorageClasses and configure this behavior without writing raw YAML manifests.

## Prerequisites

- Portainer BE or CE with a Kubernetes environment connected
- A storage provisioner installed in your cluster (e.g., local-path, NFS, Longhorn, OpenEBS)
- Admin access to Portainer

## Understanding StorageClasses

A StorageClass defines:
- **Provisioner**: Which plugin creates the volume (e.g., `rancher.io/local-path`)
- **Reclaim Policy**: `Delete` or `Retain` - what happens to the PV when the PVC is deleted
- **Volume Binding Mode**: `Immediate` or `WaitForFirstConsumer`

## Step 1: Verify a Provisioner Is Installed

Before configuring dynamic provisioning, ensure a provisioner runs in your cluster:

```bash
# Check for storage provisioner pods

kubectl get pods -A | grep -E "(provisioner|local-path|nfs|longhorn)"

# Check available StorageClasses
kubectl get storageclass
```

## Step 2: Create a StorageClass in Portainer

### Via Portainer UI

1. Log into Portainer.
2. Select your **Kubernetes** environment.
3. Navigate to **Cluster** → **Storage**.
4. Click **Add storage class**.
5. Fill in:
   - **Name**: `fast-ssd`
   - **Provisioner**: `kubernetes.io/no-provisioner` (for local) or your cloud provisioner
   - **Reclaim policy**: `Delete`
   - **Volume binding mode**: `WaitForFirstConsumer`
6. Click **Create storage class**.

### Via kubectl (applied through Portainer's KubeShell)

```yaml
# storageclass-local-path.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # Set as default
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```bash
# Apply via KubeShell in Portainer
kubectl apply -f storageclass-local-path.yaml
```

## Step 3: Set a Default StorageClass

```bash
# Set an existing StorageClass as the default
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get storageclass
# NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      DEFAULT
# local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   yes
```

## Step 4: Configure Dynamic Provisioning in Portainer

In Portainer's Kubernetes settings:

1. Go to **Cluster** → **Setup**.
2. Under **Storage**, enable **Allow users to use the default storage class when deploying applications**.
3. This lets non-admin users create PVCs without specifying a StorageClass.

## Step 5: Deploy an Application with a PVC

When deploying via Portainer's **Applications** UI:

1. Go to **Applications** → **Add application**.
2. Select your namespace and deployment type.
3. In the **Persisting data** section, click **Add a new persistent volume**.
4. Configure:
   - **Persistence type**: New persistent volume
   - **Storage class**: Select from dropdown (shows your configured StorageClasses)
   - **Size**: e.g., `5Gi`
   - **Access mode**: `ReadWriteOnce`
   - **Mount path**: `/data`
5. Deploy the application. Portainer automatically creates the PVC and the provisioner creates the PV.

## Step 6: Verify Provisioning

```bash
# Check PVCs in namespace
kubectl get pvc -n your-namespace

# Check that PV was dynamically created
kubectl get pv

# Example output showing dynamic provisioning:
# NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   STORAGECLASS
# pvc-a1b2c3d4-e5f6-7890-abcd-ef1234567890   5Gi        RWO            Delete           Bound    local-path
```

## Configuring NFS Dynamic Provisioning

For shared storage with `ReadWriteMany` access:

```yaml
# nfs-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  server: 192.168.1.100       # NFS server IP
  path: /exported/path         # NFS export path
  archiveOnDelete: "false"     # Delete data when PVC is removed
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

## Conclusion

Dynamic provisioning in Kubernetes eliminates the need to manually pre-provision PersistentVolumes. By configuring StorageClasses in Portainer, both administrators and developers can request storage on demand through PVCs. Choose the right provisioner for your environment, set sensible reclaim policies, and configure a default StorageClass to simplify the developer experience.
