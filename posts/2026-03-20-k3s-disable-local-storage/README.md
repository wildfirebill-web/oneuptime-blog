# How to Disable Local Storage in K3s

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kubernetes, Rancher, Storage, Local Path Provisioner, Longhorn

Description: Learn how to disable K3s's built-in local path provisioner and replace it with a production storage solution like Longhorn or NFS.

## Introduction

K3s includes the Local Path Provisioner by default, which dynamically creates PersistentVolumes as directories on the local node's filesystem. While this is great for development, it has a critical limitation for production: data is node-local. If a pod is rescheduled to a different node, it loses access to its PersistentVolume. For production deployments, distributed storage solutions like Longhorn, NFS, or Rook/Ceph are required.

## Understanding Local Path Provisioner

The local path provisioner:
- Creates PVs as directories under `/var/lib/rancher/k3s/storage/` by default
- Is simple and requires no external infrastructure
- Does NOT provide data redundancy or replication
- Data is tied to the specific node where the PVC was first created

## When to Disable Local Storage

- Production workloads that require data persistence across pod rescheduling
- Multi-node clusters where pods can migrate
- Stateful applications requiring replicated storage (databases, etc.)
- When using an external storage solution (NFS, Longhorn, Ceph, etc.)

## Disabling Local Storage

### Before Installation

```bash
sudo mkdir -p /etc/rancher/k3s

sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
token: "ClusterToken"
disable:
  - local-storage
EOF

curl -sfL https://get.k3s.io | sudo sh -
```

### On an Existing Cluster

```bash
# Add local-storage to the disable list

echo "  - local-storage" >> /etc/rancher/k3s/config.yaml

# Restart K3s
sudo systemctl restart k3s

# Verify the local-path provisioner is removed
kubectl get pods -n kube-system | grep local-path
# Should return empty

# Check that no local-path StorageClass exists
kubectl get storageclass
```

## Option 1: Install Longhorn (Recommended for Production)

Longhorn provides replicated block storage with a web UI:

### Prerequisites

```bash
# Install required packages on each node
sudo apt-get install -y open-iscsi nfs-common util-linux

# Enable open-iscsi
sudo systemctl enable iscsid
sudo systemctl start iscsid

# Verify
sudo systemctl status iscsid
```

### Install Longhorn

```bash
# Install using Helm
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
    --namespace longhorn-system \
    --create-namespace \
    --set defaultSettings.defaultReplicaCount=2 \
    --set defaultSettings.storageMinimalAvailablePercentage=10 \
    --set persistence.defaultClassReplicaCount=2

# Wait for all Longhorn pods to be ready
kubectl -n longhorn-system rollout status deployment/longhorn-driver-deployer
kubectl -n longhorn-system get pods

# Verify the Longhorn StorageClass is created
kubectl get storageclass
# longhorn should appear as the default
```

### Access the Longhorn UI

```bash
# Port-forward to the Longhorn UI
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80 &

# Access at http://localhost:8080
```

### Test Longhorn Storage

```yaml
# longhorn-pvc-test.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: longhorn-test
spec:
  containers:
    - name: test
      image: alpine
      command: ["sh", "-c", "while true; do date >> /data/test.txt; sleep 5; done"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: longhorn-test-pvc
```

```bash
kubectl apply -f longhorn-pvc-test.yaml
kubectl exec -it longhorn-test -- cat /data/test.txt
kubectl delete pod longhorn-test
```

## Option 2: Install NFS Storage Provisioner

For organizations with an existing NFS server:

```bash
# Install NFS client utilities on all K3s nodes
sudo apt-get install -y nfs-common

# Install the NFS CSI driver
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update

helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
    --namespace kube-system

# Create a StorageClass for NFS
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.example.com    # Your NFS server IP or hostname
  share: /exports/k3s               # NFS export path
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.1
EOF
```

### Test NFS Storage

```yaml
# nfs-pvc-test.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-test-pvc
spec:
  accessModes:
    - ReadWriteMany    # NFS supports ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 1Gi
```

## Option 3: OpenEBS Local PV

For a more advanced local storage solution with snapshot support:

```bash
# Install OpenEBS
helm repo add openebs https://openebs.github.io/charts
helm repo update

helm install openebs openebs/openebs \
    --namespace openebs \
    --create-namespace \
    --set legacy.enabled=false \
    --set localprovisioner.enabled=true \
    --set ndm.enabled=false

# Create a default StorageClass
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-hostpath
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: openebs.io/local
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF
```

## Setting a New Default StorageClass

After installing a replacement storage provider, ensure it's set as the default:

```bash
# Check current default
kubectl get storageclass | grep "(default)"

# Set a new storage class as default
kubectl patch storageclass longhorn \
    -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Remove the default annotation from any old storage class
kubectl patch storageclass local-path \
    -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}' \
    2>/dev/null || true
```

## Conclusion

Disabling K3s's local storage provisioner is the first step toward production-grade persistent storage. Longhorn is the recommended replacement for K3s clusters, offering replicated block storage with excellent Kubernetes integration and a management UI. For organizations with existing NFS infrastructure, the NFS CSI driver provides a straightforward path to shared persistent volumes. Regardless of which storage solution you choose, ensure it provides data replication to prevent data loss when nodes fail.
