# How to Set Up Local Path Provisioner for Development in Rancher (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Local Path Provisioner, Storage, Development

Description: Configure the Local Path Provisioner in Rancher for lightweight persistent storage in development environments without requiring cloud storage backends.

## Introduction

The Local Path Provisioner, developed by Rancher Labs, provides a simple way to use local disk storage in Kubernetes clusters. It automatically creates host path-based PersistentVolumes for development workloads. Unlike cloud storage providers, it requires zero external dependencies, making it ideal for development clusters, edge deployments, and single-node setups.

## Prerequisites

- Rancher-managed Kubernetes cluster
- kubectl access with cluster-admin permissions
- Local disk space on worker nodes

## Step 1: Install Local Path Provisioner

```bash
# Install using the official manifest

kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml

# Verify installation
kubectl get pods -n local-path-storage
kubectl get storageclass local-path

# Check the storage class details
kubectl describe storageclass local-path
```

## Step 2: Make Local Path the Default StorageClass

```bash
# Set local-path as the default StorageClass
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# If another StorageClass is the default, remove that annotation first
kubectl patch storageclass standard \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Verify
kubectl get storageclass
```

## Step 3: Configure Storage Paths

```yaml
# local-path-config.yaml - Customize storage paths
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-path-config
  namespace: local-path-storage
data:
  config.json: |-
    {
      "nodePathMap": [
        {
          "node": "DEFAULT_PATH_FOR_NON_LISTED_NODES",
          "paths": ["/opt/local-path-provisioner"]
        },
        {
          "node": "worker-node-01",
          "paths": ["/data/fast-ssd", "/data/slow-hdd"]
        },
        {
          "node": "worker-node-02",
          "paths": ["/data/storage"]
        }
      ]
    }
  setup: |-
    #!/bin/sh
    set -eu
    mkdir -m 0777 -p "$VOL_DIR"
  teardown: |-
    #!/bin/sh
    set -eu
    rm -rf "$VOL_DIR"
  helperPod.yaml: |-
    apiVersion: v1
    kind: Pod
    metadata:
      name: helper-pod
    spec:
      containers:
      - name: helper-pod
        image: busybox
        imagePullPolicy: IfNotPresent
```

## Step 4: Create PersistentVolumeClaims

```yaml
# dev-pvc.yaml - PVC for development database
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: development
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 10Gi
---
# redis-pvc.yaml - PVC for Redis
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-data
  namespace: development
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
```

## Step 5: Deploy Stateful Application

```yaml
# postgres-dev.yaml - PostgreSQL with local path storage
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: development
spec:
  serviceName: postgresql
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql
          image: postgres:15
          env:
            - name: POSTGRES_DB
              value: devdb
            - name: POSTGRES_USER
              value: developer
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path
        resources:
          requests:
            storage: 10Gi
```

## Step 6: Inspect Provisioned Volumes

```bash
# List PersistentVolumes
kubectl get pv

# Describe a specific PV to find host path
kubectl describe pv pvc-<uuid>

# Check what's stored on the node
# SSH into the worker node
ssh worker-node-01
ls /opt/local-path-provisioner/

# List directories created by provisioner
ls /opt/local-path-provisioner/pvc-*/
```

## Step 7: Backup and Restore

```bash
# Backup using rsync (run on the node)
rsync -avz /opt/local-path-provisioner/pvc-<uuid>/ \
  backup-host:/backups/postgres-$(date +%Y%m%d)/

# Or create a backup pod
kubectl run backup-job \
  --image=alpine \
  --restart=Never \
  --rm \
  -it \
  --overrides='{
    "spec": {
      "volumes": [{
        "name": "data",
        "persistentVolumeClaim": {"claimName": "postgres-data"}
      }],
      "containers": [{
        "name": "backup",
        "image": "alpine",
        "command": ["tar", "czf", "/backup/data.tar.gz", "/data"],
        "volumeMounts": [{"name": "data", "mountPath": "/data"}]
      }]
    }
  }' \
  --namespace=development
```

## Step 8: Reclaim Policy Configuration

```yaml
# Retain data when PVC is deleted (useful for development)
kubectl patch storageclass local-path \
  -p '{"reclaimPolicy": "Retain"}'

# Or create a custom StorageClass with Retain policy
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path-retain
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

## Conclusion

The Local Path Provisioner provides a zero-dependency storage solution for development environments in Rancher. Its simplicity and tight integration with Rancher makes it the default choice for K3s and development clusters. While not suitable for production multi-node workloads requiring shared storage, it excels for development databases, caches, and stateful services where data locality is acceptable and fast local disk I/O is beneficial.
