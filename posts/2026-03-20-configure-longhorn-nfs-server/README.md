# How to Configure Longhorn Network File System Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, NFS, Share Manager, RWX

Description: Configure Longhorn's built-in NFS server (share manager) settings for ReadWriteMany volumes, including performance tuning and network configuration.

## Introduction

Longhorn implements ReadWriteMany (RWX) volumes using an internal NFSv4.1 server called the Share Manager. Each RWX volume gets its own Share Manager pod that acts as an NFS server, allowing multiple pods across different nodes to mount the same volume. This guide covers configuring the Share Manager settings and tuning the NFS server behavior.

## How Longhorn's Share Manager Works

```text
Pod A (Node 1) ──NFSv4.1──┐
                           ├──→ Share Manager Pod ──→ Longhorn Volume (Block)
Pod B (Node 2) ──NFSv4.1──┘
```

The Share Manager pod:
- Runs on the same node as the Longhorn volume it serves
- Uses in-kernel NFS server (or NFS-Ganesha depending on configuration)
- Creates a Kubernetes Service for the NFS endpoint
- Provides NFSv4.1 access to all pods in the cluster

## Prerequisites

- Longhorn v1.1.0 or later installed
- `nfs-common` on Ubuntu/Debian or `nfs-utils` on RHEL/CentOS on all nodes

```bash
# Ubuntu/Debian

apt-get install -y nfs-common

# RHEL/CentOS
yum install -y nfs-utils

# Verify NFS client modules
lsmod | grep nfs
```

## Configuring Share Manager Image

The Share Manager uses a dedicated container image. Configure which image is used:

```bash
# Check the current Share Manager image setting
kubectl get settings.longhorn.io share-manager-image \
  -n longhorn-system -o yaml

# Update the Share Manager image (usually done during upgrades)
kubectl patch settings.longhorn.io share-manager-image \
  -n longhorn-system \
  --type merge \
  -p '{"value": "longhornio/longhorn-share-manager:v1.7.0"}'
```

## Configuring Share Manager Tolerations

If your nodes have taints, configure tolerations for Share Manager pods:

```bash
# Set tolerations for Share Manager pods (same format as Longhorn tolerations)
kubectl patch settings.longhorn.io taint-toleration \
  -n longhorn-system \
  --type merge \
  -p '{"value": "dedicated=storage:NoSchedule"}'
```

## Creating an RWX Volume

```yaml
# rwx-pvc-nfs.yaml - ReadWriteMany PVC using Longhorn's NFS share manager
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-web-content
  namespace: default
spec:
  accessModes:
    - ReadWriteMany    # Triggers Share Manager creation
  storageClassName: longhorn
  resources:
    requests:
      storage: 20Gi
```

```bash
kubectl apply -f rwx-pvc-nfs.yaml

# Watch the Share Manager pod being created
kubectl get pods -n longhorn-system -l app=longhorn-share-manager -w
```

## Checking Share Manager Status

```bash
# List all Share Manager pods
kubectl get pods -n longhorn-system \
  -l app=longhorn-share-manager \
  -o wide

# Check Share Manager logs
kubectl logs -n longhorn-system \
  -l app=longhorn-share-manager \
  --tail=50

# Check the NFS services
kubectl get services -n longhorn-system | grep share
```

## Configuring NFS Mount Options

Customize how clients mount the NFS share by specifying mount options in the StorageClass:

```yaml
# storageclass-rwx-tuned.yaml - RWX StorageClass with optimized NFS options
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-rwx-optimized
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "3"
  fsType: "ext4"
# NFS mount options for better performance
mountOptions:
  - vers=4.1      # Use NFSv4.1 for better performance
  - noresvport    # Don't require reserved ports
  - hard          # Retry on failure (important for reliability)
  - intr          # Allow interrupting hung NFS operations
  - noacl         # Disable ACL (performance improvement)
  - noatime       # Don't update access times (performance)
```

```bash
kubectl apply -f storageclass-rwx-tuned.yaml
```

## Testing RWX Functionality

```yaml
# rwx-test.yaml - Test that RWX volumes work across multiple pods
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rwx-test-app
  namespace: default
spec:
  replicas: 3   # All 3 pods share the same volume
  selector:
    matchLabels:
      app: rwx-test
  template:
    metadata:
      labels:
        app: rwx-test
    spec:
      containers:
        - name: writer
          image: busybox
          command:
            - sh
            - -c
            - |
              while true; do
                echo "$(hostname): $(date)" >> /shared/log.txt
                sleep 5
              done
          volumeMounts:
            - name: shared
              mountPath: /shared
      volumes:
        - name: shared
          persistentVolumeClaim:
            claimName: shared-web-content
```

```bash
kubectl apply -f rwx-test.yaml

# Wait for pods to start
kubectl get pods -l app=rwx-test

# Check that all pods are writing to the same file
kubectl exec -it \
  $(kubectl get pod -l app=rwx-test -o name | head -1) \
  -- tail -20 /shared/log.txt
# Should show log lines from all 3 pods
```

## Share Manager High Availability

The Share Manager pod runs on a specific node. If that node fails, Longhorn reschedules it:

```bash
# Simulate Share Manager pod failure
kubectl delete pod -n longhorn-system \
  $(kubectl get pods -n longhorn-system -l app=longhorn-share-manager -o name | head -1)

# Observe recovery - Longhorn creates a new Share Manager pod
kubectl get pods -n longhorn-system -l app=longhorn-share-manager -w
```

During the pod restart, pods accessing the volume via NFS may experience a brief interruption. NFSv4.1's state recovery mechanism typically handles this transparently.

## Setting Share Manager Priority Class

```bash
# Give Share Manager pods high priority to prevent eviction
kubectl patch settings.longhorn.io priority-class \
  -n longhorn-system \
  --type merge \
  -p '{"value": "system-node-critical"}'
```

## Monitoring Share Manager Performance

```bash
# Check NFS statistics inside the Share Manager
kubectl exec -it -n longhorn-system \
  $(kubectl get pods -n longhorn-system -l app=longhorn-share-manager -o name | head -1) \
  -- nfsstat -s 2>/dev/null || cat /proc/net/rpc/nfsd

# Monitor Share Manager CPU/memory usage
kubectl top pods -n longhorn-system | grep share-manager
```

## Conclusion

Longhorn's built-in NFS Share Manager provides a convenient way to implement ReadWriteMany storage without external NFS infrastructure. By understanding how to configure mount options, tolerations, and monitoring the Share Manager, you can provide reliable shared storage for web serving, content distribution, and other multi-pod read/write scenarios. For workloads requiring the highest NFS performance, tune the mount options and ensure the Share Manager pod is running on the same node as the volume for minimum latency.
