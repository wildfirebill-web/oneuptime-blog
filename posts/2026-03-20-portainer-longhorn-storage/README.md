# How to Deploy Longhorn Storage and Manage via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Longhorn, Kubernetes, Storage, Persistent Volumes

Description: Deploy Longhorn distributed block storage on Kubernetes and manage volumes, backups, and replicas through both Longhorn UI and Portainer.

## Introduction

Longhorn is a cloud-native distributed block storage system for Kubernetes. It provides persistent volumes with automatic replication, snapshots, and backup capabilities. Portainer can manage the Kubernetes workloads that consume Longhorn volumes, while Longhorn's own UI handles volume-specific operations.

## Prerequisites

- Kubernetes cluster managed by Portainer
- Minimum 3 worker nodes for replication
- `open-iscsi` installed on all nodes
- `kubectl` access

## Step 1: Install Prerequisites on All Nodes

```bash
# Install open-iscsi on all Kubernetes nodes

# Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y open-iscsi

# RHEL/CentOS
sudo yum install -y iscsi-initiator-utils

# Enable and start
sudo systemctl enable --now iscsid

# Verify
sudo systemctl status iscsid

# Check Longhorn environment (run on one node)
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/scripts/environment_check.sh | bash
```

## Step 2: Install Longhorn via Helm

```bash
# Add Longhorn Helm repository
helm repo add longhorn https://charts.longhorn.io
helm repo update

# Install Longhorn
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --version 1.6.0 \
  --set defaultSettings.defaultReplicaCount=3 \
  --set defaultSettings.defaultDataPath=/var/lib/longhorn

# Wait for Longhorn to be ready
kubectl -n longhorn-system rollout status deploy/longhorn-driver-deployer
kubectl -n longhorn-system get pods
```

## Step 3: Access Longhorn UI via Portainer

```bash
# Create an Ingress for Longhorn UI via kubectl
kubectl apply -f - << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: longhorn-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
spec:
  ingressClassName: nginx
  rules:
  - host: longhorn.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
EOF

# Or use port-forward for quick access
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
```

## Step 4: Create Storage Classes

```yaml
# Create a Portainer-specific StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-portainer
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  fromBackup: ""
  fsType: "ext4"
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

## Step 5: Deploy Applications Using Longhorn via Portainer

In Portainer's Kubernetes interface, deploy a stateful application:

```yaml
# stateful-app.yml - deploy via Portainer
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: default
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: "secretpassword"
        volumeMounts:
        - name: pgdata
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: pgdata
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: longhorn-portainer
      resources:
        requests:
          storage: 10Gi
```

## Step 6: Create Snapshots and Backups

```bash
# Create a snapshot via Longhorn API
curl -X POST \
  http://longhorn.example.com/v1/volumes/pvc-xxx/actions/snapshotCreate \
  -H "Content-Type: application/json" \
  -d '{"name": "before-upgrade-2024"}'

# Configure backup target (S3)
kubectl apply -f - << 'EOF'
apiVersion: longhorn.io/v1beta2
kind: Setting
metadata:
  name: backup-target
  namespace: longhorn-system
value: "s3://my-bucket@us-east-1/longhorn"
---
apiVersion: longhorn.io/v1beta2
kind: Setting
metadata:
  name: backup-target-credential-secret
  namespace: longhorn-system
value: "longhorn-s3-secret"
EOF
```

## Volume Management via Portainer

In Portainer's Kubernetes interface:

1. Go to **Kubernetes > Volumes**
2. View PVCs and their Longhorn backing volumes
3. Expand volumes: Edit PVC, increase storage size
4. Monitor usage: Volumes > PVC > Details

```bash
# Expand a Longhorn volume
kubectl patch pvc pgdata-postgres-0 \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

## Conclusion

Longhorn provides enterprise-grade persistent storage for Kubernetes workloads managed by Portainer. Its built-in replication, snapshots, and backup capabilities make it ideal for production stateful workloads. Portainer handles the application lifecycle while Longhorn handles the storage lifecycle-a clean separation of concerns that simplifies operations.
