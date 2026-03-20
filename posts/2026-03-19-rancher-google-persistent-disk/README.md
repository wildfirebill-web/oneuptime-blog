# How to Configure Google Persistent Disk in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Storage, GCP

Description: A practical guide to configuring Google Persistent Disk storage for Rancher-managed Kubernetes clusters on Google Cloud.

Google Persistent Disk provides durable, high-performance block storage for Kubernetes workloads on Google Cloud Platform. Rancher supports GCE PD through the GCE PD CSI driver, enabling dynamic provisioning, snapshots, and regional replication. This guide covers the complete setup.

## Prerequisites

- A running Rancher instance
- A GCP-based Kubernetes cluster (GKE or RKE on GCE VMs)
- GCP service account with appropriate permissions
- kubectl and Helm access to your cluster

## Step 1: Configure IAM Permissions

Create a service account with the required permissions:

```bash
gcloud iam service-accounts create gce-pd-csi-sa \
  --display-name "GCE PD CSI Driver"

gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member "serviceAccount:gce-pd-csi-sa@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role "roles/compute.storageAdmin"

gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member "serviceAccount:gce-pd-csi-sa@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role "roles/iam.serviceAccountUser"
```

## Step 2: Install the GCE PD CSI Driver

For GKE clusters, the CSI driver is pre-installed. For RKE clusters on GCE:

```bash
helm repo add gce-pd-csi-driver https://kubernetes-sigs.github.io/gcp-compute-persistent-disk-csi-driver/charts
helm repo update

helm install gce-pd-csi-driver gce-pd-csi-driver/gce-pd-csi-driver \
  --namespace kube-system
```

Verify:

```bash
kubectl get pods -n kube-system -l app=gcp-compute-persistent-disk-csi-driver
kubectl get csidrivers | grep pd.csi.storage.gke.io
```

## Step 3: Create Storage Classes

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-standard
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-standard
  fstype: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  fstype: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-balanced
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-balanced
  fstype: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-extreme
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-extreme
  fstype: ext4
  provisioned-iops-on-create: "10000"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

```bash
kubectl apply -f gcp-storageclasses.yaml
```

## Step 4: Create a PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gcp-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: pd-ssd
  resources:
    requests:
      storage: 30Gi
```

```bash
kubectl apply -f gcp-pvc.yaml
```

## Step 5: Deploy an Application

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: default
spec:
  serviceName: mongodb
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:7
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-data
          mountPath: /data/db
  volumeClaimTemplates:
  - metadata:
      name: mongo-data
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: pd-ssd
      resources:
        requests:
          storage: 50Gi
```

## Step 6: Configure Regional Persistent Disks

Regional PDs replicate data across two zones for high availability:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-ssd-regional
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
allowedTopologies:
- matchLabelExpressions:
  - key: topology.gke.io/zone
    values:
    - us-central1-a
    - us-central1-b
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

## Step 7: Configure Volume Snapshots

Create a VolumeSnapshotClass:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: gce-snapshot-class
driver: pd.csi.storage.gke.io
deletionPolicy: Retain
```

Take a snapshot:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mongo-snapshot
  namespace: default
spec:
  volumeSnapshotClassName: gce-snapshot-class
  source:
    persistentVolumeClaimName: mongo-data-mongodb-0
```

Restore from snapshot:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-mongo-data
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: pd-ssd
  resources:
    requests:
      storage: 50Gi
  dataSource:
    name: mongo-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

## Step 8: Configure CMEK Encryption

Use Customer-Managed Encryption Keys:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-encrypted
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  disk-encryption-kms-key: projects/<project>/locations/<region>/keyRings/<ring>/cryptoKeys/<key>
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

## Step 9: Configure ReadOnlyMany Volumes

Create a snapshot and use it as a read-only volume for multiple pods:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-readonly
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
volumeBindingMode: Immediate
```

Mount as read-only:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: reader-pod
spec:
  containers:
  - name: reader
    image: nginx:latest
    volumeMounts:
    - name: data
      mountPath: /data
      readOnly: true
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: gcp-pvc
      readOnly: true
```

## Step 10: Monitor Google Persistent Disks

```bash
# Check PVCs

kubectl get pvc --all-namespaces

# View PV details
kubectl describe pv <pv-name>

# Check CSI driver
kubectl get pods -n kube-system -l app=gcp-compute-persistent-disk-csi-driver

# Check driver logs
kubectl logs -n kube-system -l app=gcp-compute-persistent-disk-csi-driver --tail=50

# List disks via gcloud
gcloud compute disks list --filter="name~pvc"
```

## Troubleshooting

- **PVC Pending**: Verify IAM permissions and CSI driver status
- **Zone mismatch**: Use `WaitForFirstConsumer` and ensure nodes exist in the target zone
- **Quota exceeded**: Check GCP disk quota in the project
- **Attach limit**: GCE instances have a maximum number of attachable disks
- **Regional PD errors**: Ensure both zones have nodes available

## Summary

Google Persistent Disk in Rancher provides versatile block storage options for Kubernetes workloads on GCP. With support for standard, SSD, balanced, and extreme disk types, plus regional replication for high availability, you can match storage performance to workload requirements. The GCE PD CSI driver enables dynamic provisioning, snapshots, and volume expansion, making storage management seamless in your Rancher-managed clusters.
