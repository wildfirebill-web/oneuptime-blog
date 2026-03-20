# How to Set Up Harvester Storage for Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Storage, Kubernetes, Longhorn, CSI, PersistentVolume, SUSE Rancher, HCI

Description: Learn how to configure Harvester's built-in Longhorn storage for Kubernetes clusters provisioned on Harvester, including StorageClasses, PVC creation, and storage topology for VM-based workloads.

---

Harvester uses Longhorn as its built-in storage backend, providing hyperconverged storage that is shared between virtual machines and Kubernetes workloads. Kubernetes clusters provisioned on Harvester can consume this storage directly via CSI.

---

## Storage Architecture in Harvester

```
┌─────────────────────────────────────────┐
│           Harvester Cluster             │
│                                         │
│  ┌─────────┐      ┌──────────────────┐  │
│  │  VMs    │      │  K8s Clusters    │  │
│  │  (QCOW) │      │  (RKE2 / K3s)   │  │
│  └────┬────┘      └────────┬─────────┘  │
│       │                    │            │
│       └──────────┬─────────┘            │
│                  │                      │
│           ┌──────▼──────┐               │
│           │  Longhorn   │               │
│           │  (Storage)  │               │
│           └─────────────┘               │
└─────────────────────────────────────────┘
```

---

## Step 1: Verify Longhorn is Running on Harvester

```bash
# Connect to the Harvester management cluster
export KUBECONFIG=/path/to/harvester-kubeconfig

# Check Longhorn is running
kubectl get pods -n longhorn-system | head -20

# Check available storage nodes
kubectl get node.longhorn.io -n longhorn-system
```

---

## Step 2: Review Harvester Default StorageClasses

```bash
# List available StorageClasses
kubectl get storageclass

# Harvester provides these by default:
# harvester-longhorn (default) — 3 replicas
# longhorn — standard Longhorn StorageClass
```

---

## Step 3: Create a Custom StorageClass for Kubernetes Workloads

```yaml
# k8s-workloads-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: harvester-ssd
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
  dataLocality: "best-effort"
  diskSelector: "ssd"           # Target SSDs if tagged in Longhorn
  fsType: "ext4"
```

---

## Step 4: Provision K8s Clusters on Harvester via Rancher

When creating an RKE2 or K3s cluster on Harvester through Rancher, configure the cloud provider to use Harvester storage:

```yaml
# In the Rancher cluster provisioning YAML, configure the Harvester CSI
# This is typically done through the Rancher UI under:
# Cluster Management → Create → Harvester → Cloud Provider: Harvester

# The Harvester cloud provider installs automatically and creates:
# - harvester-longhorn StorageClass
# - The CSI driver for dynamic provisioning
```

---

## Step 5: Create PVCs on the Guest Kubernetes Cluster

Once the K8s cluster is running on Harvester VMs, create PVCs:

```yaml
# database-pvc.yaml (on the guest Kubernetes cluster)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: harvester-longhorn   # Use the Harvester storage class
  resources:
    requests:
      storage: 50Gi
```

```bash
kubectl apply -f database-pvc.yaml

# Verify PVC is bound
kubectl get pvc -n production
```

---

## Step 6: Configure Storage Topology for Multi-Node K8s on Harvester

For multi-node Kubernetes clusters on Harvester, ensure volumes are placed on nodes where the VMs are running:

```yaml
# StorageClass with WaitForFirstConsumer binding mode
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: harvester-local
provisioner: driver.longhorn.io
volumeBindingMode: WaitForFirstConsumer   # Wait until pod is scheduled
reclaimPolicy: Delete
parameters:
  numberOfReplicas: "2"
  dataLocality: "strict-local"
```

---

## Step 7: Monitor Storage Usage from Harvester

```bash
# Check Longhorn volume status from the Harvester cluster
kubectl get volume -n longhorn-system

# Check storage capacity per node
kubectl get node.longhorn.io -n longhorn-system \
  -o custom-columns='NODE:.metadata.name,STORAGE:.status.diskStatus'

# Access the Longhorn UI (from Harvester dashboard)
# Navigate to: Harvester → More → Longhorn
```

---

## Step 8: Configure Backup Target for Guest Cluster PVCs

```bash
# Set Longhorn backup target (from the Harvester management cluster)
kubectl patch setting -n longhorn-system \
  backup-target \
  --type merge \
  -p '{"value":"s3://my-backups@us-west-2/harvester-longhorn"}'

# Create volume backups from the guest cluster
kubectl apply -f - <<EOF
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: daily-backup
  namespace: longhorn-system
spec:
  cron: "0 2 * * *"
  task: "backup"
  groups: ["default"]
  retain: 7
EOF
```

---

## Best Practices

- Use `WaitForFirstConsumer` volume binding mode for Kubernetes clusters on Harvester — this ensures volumes are created on the same Harvester node as the VM, reducing cross-node storage traffic.
- Size Harvester node disks generously — Longhorn volumes for both the VMs and the guest Kubernetes workloads compete for the same physical storage.
- Tag SSDs and HDDs differently in Longhorn (`disk-type=ssd`, `disk-type=hdd`) and create separate StorageClasses for each — this lets you route database workloads to SSDs and archival workloads to HDDs.
