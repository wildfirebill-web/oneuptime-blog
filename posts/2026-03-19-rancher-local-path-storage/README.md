# How to Configure Local Path Storage in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Storage, Persistent Volumes

Description: Learn how to configure Local Path Provisioner in Rancher for dynamic local storage provisioning on Kubernetes nodes.

Local Path Provisioner enables dynamic provisioning of local storage on Kubernetes nodes. It is the default storage provisioner for K3s clusters and provides a simple way to use node-local storage for development and lightweight production workloads. This guide covers configuration and usage in Rancher.

## Prerequisites

- A running Rancher instance (v2.6 or later)
- A managed Kubernetes cluster
- kubectl access to your cluster

## Understanding Local Path Storage

Local Path Provisioner creates hostPath-based persistent volumes on the node where the pod is scheduled. Key characteristics:

- Storage is tied to a specific node
- Data does not survive node failure
- Provides the best performance since there is no network overhead
- Supports ReadWriteOnce access mode only

## Step 1: Check for Existing Local Path Provisioner

K3s clusters include Local Path Provisioner by default:

```bash
kubectl get storageclass
kubectl get pods -n kube-system | grep local-path
```

If you see a `local-path` StorageClass, it is already available.

## Step 2: Install Local Path Provisioner

For clusters without it, install using kubectl:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml
```

Verify the installation:

```bash
kubectl get pods -n local-path-storage
kubectl get storageclass local-path
```

## Step 3: Install via Helm

```bash
helm repo add local-path-provisioner https://github.com/rancher/local-path-provisioner/releases
helm repo update

helm install local-path-provisioner local-path-provisioner/local-path-provisioner \
  --namespace local-path-storage \
  --create-namespace \
  --set storageClass.defaultClass=true
```

## Step 4: Configure the Storage Path

By default, volumes are created under `/opt/local-path-provisioner`. Customize this by editing the ConfigMap:

```bash
kubectl edit configmap local-path-config -n local-path-storage
```

Modify the config:

```json
{
  "nodePathMap": [
    {
      "node": "DEFAULT_PATH_FOR_NON_LISTED_NODES",
      "paths": ["/mnt/fast-storage/local-path"]
    },
    {
      "node": "worker-node-1",
      "paths": ["/mnt/ssd/local-path"]
    },
    {
      "node": "worker-node-2",
      "paths": ["/mnt/nvme/local-path", "/mnt/ssd/local-path"]
    }
  ]
}
```

This allows you to specify different storage paths per node.

## Step 5: Create a PVC with Local Path

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f local-pvc.yaml
```

The PVC will remain in Pending state until a pod uses it (WaitForFirstConsumer binding mode).

## Step 6: Deploy an Application with Local Path Storage

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: local-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: local-app
  template:
    metadata:
      labels:
        app: local-app
    spec:
      containers:
      - name: app
        image: nginx:latest
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: local-pvc
```

```bash
kubectl apply -f local-app.yaml
kubectl get pvc local-pvc
```

The PVC should now be Bound.

## Step 7: Use with StatefulSets

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cache
  namespace: default
spec:
  serviceName: cache
  replicas: 3
  selector:
    matchLabels:
      app: cache
  template:
    metadata:
      labels:
        app: cache
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: cache-data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: cache-data
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: local-path
      resources:
        requests:
          storage: 5Gi
```

Each StatefulSet replica gets its own local volume on its scheduled node.

## Step 8: Configure Reclaim Policy

By default, Local Path Provisioner uses the `Delete` reclaim policy. Change to `Retain` if needed:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path-retain
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

## Step 9: Set Up Node Affinity

Since local storage is node-specific, use node affinity to ensure pods are scheduled on the correct node:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pinned-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pinned-app
  template:
    metadata:
      labels:
        app: pinned-app
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - worker-node-1
      containers:
      - name: app
        image: nginx:latest
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: local-pvc
```

## Step 10: Monitor Local Path Storage

```bash
# Check storage class
kubectl get storageclass local-path

# List PVCs using local-path
kubectl get pvc --all-namespaces -o custom-columns='NAME:.metadata.name,CLASS:.spec.storageClassName,STATUS:.status.phase' | grep local-path

# Check provisioner logs
kubectl logs -n local-path-storage -l app=local-path-provisioner

# Check disk usage on a node (SSH to the node)
df -h /opt/local-path-provisioner

# Check which node a PV is on
kubectl get pv -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]'
```

## Troubleshooting

- **PVC stuck in Pending**: Normal behavior with `WaitForFirstConsumer`; deploy a pod to trigger binding
- **No space left**: Check disk usage on the node's storage path
- **Pod not scheduling**: Node where PV exists might be down or cordoned
- **Data lost after pod restart**: Verify the PVC is still bound and the node is the same
- **Permission errors**: Check directory permissions on the storage path

## Summary

Local Path Provisioner in Rancher provides a simple and performant storage solution for workloads that do not require shared access or cross-node data availability. It is ideal for development environments, caches, and single-node databases. For production use, be aware that data is tied to a specific node and plan for backup and recovery accordingly.
