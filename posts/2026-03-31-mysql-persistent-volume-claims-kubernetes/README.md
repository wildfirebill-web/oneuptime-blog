# How to Use Persistent Volume Claims for MySQL on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Kubernetes, Storage, StatefulSet, PersistentVolume

Description: Configure Persistent Volume Claims for MySQL on Kubernetes to ensure data survives pod restarts and rescheduling using StatefulSets and StorageClasses.

---

MySQL requires persistent storage in Kubernetes so data is not lost when a pod restarts or is rescheduled. Kubernetes Persistent Volume Claims (PVCs) provide an abstraction that lets pods request storage without knowing the underlying implementation.

## How PVCs Work with MySQL

A PersistentVolumeClaim requests storage from a StorageClass, which provisions the actual PersistentVolume. When a MySQL pod mounts the PVC, it gets a directory backed by durable storage such as AWS EBS, GCE Persistent Disk, or a local volume. If the pod is deleted and recreated, the same PVC is reattached and the MySQL data directory is intact.

## Creating a StorageClass

For production, use a StorageClass that provisions dynamic storage. This example uses AWS EBS gp3:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mysql-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

The `Retain` reclaim policy ensures the underlying volume is not deleted when the PVC is removed - critical for database storage.

## Using volumeClaimTemplates in a StatefulSet

The recommended way to provision PVCs for MySQL is through `volumeClaimTemplates` in a StatefulSet. Each replica gets its own PVC automatically:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: mysql-storage
      resources:
        requests:
          storage: 20Gi
```

## Creating a Standalone PVC

If you need a standalone PVC outside of a StatefulSet, define it separately:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: mysql-storage
  resources:
    requests:
      storage: 20Gi
```

Then reference it in a Pod or Deployment:

```yaml
volumes:
- name: mysql-data
  persistentVolumeClaim:
    claimName: mysql-pvc
```

## Checking PVC Status

After applying the StatefulSet, verify the PVC was created and bound:

```bash
kubectl get pvc
kubectl describe pvc mysql-data-mysql-0
kubectl get pv
```

The PVC status should show `Bound`. If it stays in `Pending`, check the StorageClass provisioner and available capacity:

```bash
kubectl describe pvc mysql-data-mysql-0 | grep -A 5 Events
kubectl get storageclass
```

## Expanding a PVC

To increase the storage size of a MySQL PVC, edit the PVC if the StorageClass supports volume expansion:

```bash
kubectl patch pvc mysql-data-mysql-0 -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
kubectl get pvc mysql-data-mysql-0
```

## Summary

Using PVCs with StatefulSets is the correct approach for persistent MySQL storage on Kubernetes. The `volumeClaimTemplates` field in the StatefulSet automatically provisions one PVC per replica, ensuring each MySQL instance gets dedicated storage. Set the reclaim policy to `Retain` to protect data from accidental deletion, and choose a StorageClass that supports `ReadWriteOnce` access mode, which maps to standard block storage used by MySQL.
