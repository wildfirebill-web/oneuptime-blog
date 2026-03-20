# How to Migrate Portainer from Docker to Kubernetes - Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Docker, Migration, DevOps, Container Orchestration

Description: Learn how to move your Portainer instance from a standalone Docker deployment to a Kubernetes cluster deployment for better scalability and resilience.

---

Running Portainer on Docker is great for getting started, but deploying it on Kubernetes gives you high availability, persistent storage management, and the ability to manage it like any other workload. This guide walks through migrating your existing Portainer Docker instance to Kubernetes while preserving your data.

---

## Step 1: Back Up Your Portainer Data

Before any migration, stop Portainer and export its data volume.

```bash
# Stop Portainer on the Docker host

docker stop portainer

# Create a compressed backup of the data volume
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/portainer-k8s-migration.tar.gz -C /data .

# Verify the backup
ls -lh portainer-k8s-migration.tar.gz
```

---

## Step 2: Create a PersistentVolumeClaim in Kubernetes

Portainer needs persistent storage on Kubernetes. Create a PVC using your cluster's default storage class.

```yaml
# portainer-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: portainer-data
  namespace: portainer
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  # Remove storageClassName to use cluster default
  storageClassName: standard
```

```bash
kubectl create namespace portainer
kubectl apply -f portainer-pvc.yaml
```

---

## Step 3: Restore Backup Data to the PVC

Use a temporary pod to restore your backup data into the PVC.

```yaml
# restore-job.yaml - temporary pod to load backup data into PVC
apiVersion: v1
kind: Pod
metadata:
  name: portainer-restore
  namespace: portainer
spec:
  restartPolicy: Never
  containers:
    - name: restore
      image: alpine
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: portainer-data
          mountPath: /data
  volumes:
    - name: portainer-data
      persistentVolumeClaim:
        claimName: portainer-data
```

```bash
kubectl apply -f restore-job.yaml

# Copy the backup file into the restore pod
kubectl cp portainer-k8s-migration.tar.gz portainer/portainer-restore:/backup.tar.gz

# Extract the backup into the PVC
kubectl exec -n portainer portainer-restore -- \
  tar xzf /backup.tar.gz -C /data

# Clean up the restore pod
kubectl delete pod portainer-restore -n portainer
```

---

## Step 4: Deploy Portainer on Kubernetes

Use the official Portainer manifest, modified to use your restored PVC.

```yaml
# portainer-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portainer
  namespace: portainer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: portainer
  template:
    metadata:
      labels:
        app: portainer
    spec:
      serviceAccountName: portainer-sa-clusteradmin
      containers:
        - name: portainer
          image: portainer/portainer-ce:latest
          ports:
            - containerPort: 9000
            - containerPort: 9443
            - containerPort: 8000
          volumeMounts:
            - name: portainer-data
              mountPath: /data
      volumes:
        - name: portainer-data
          persistentVolumeClaim:
            claimName: portainer-data
```

---

## Step 5: Create the Portainer Service

```yaml
# portainer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: portainer
  namespace: portainer
spec:
  type: LoadBalancer
  selector:
    app: portainer
  ports:
    - name: http
      port: 9000
      targetPort: 9000
    - name: https
      port: 9443
      targetPort: 9443
    - name: edge
      port: 8000
      targetPort: 8000
```

```bash
kubectl apply -f portainer-deployment.yaml -f portainer-service.yaml

# Get the external IP
kubectl get svc portainer -n portainer
```

---

## Step 6: Verify the Migration

1. Open Portainer at the LoadBalancer IP
2. Log in with your existing credentials (restored from backup)
3. Check that environments, stacks, users, and registries are all present
4. Update any agent endpoint URLs to reflect new addresses if needed

---

## Summary

Moving Portainer from Docker to Kubernetes requires backing up the data volume, restoring it into a PVC, and deploying via Kubernetes manifests. The entire configuration - users, environments, stacks, and registries - travels with the `portainer.db` file inside that volume.
