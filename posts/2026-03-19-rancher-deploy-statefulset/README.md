# How to Deploy a StatefulSet Workload in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, StatefulSet, Workloads

Description: Learn how to deploy a StatefulSet workload in Rancher for stateful applications that require stable network identities and persistent storage.

StatefulSets are designed for applications that require stable network identities, ordered deployment, and persistent storage. Unlike Deployments, each pod in a StatefulSet gets a unique, persistent identifier that is maintained across rescheduling. Common use cases include databases, message queues, and distributed systems like ZooKeeper or Cassandra. This guide walks you through deploying a StatefulSet in Rancher.

## Prerequisites

- A running Rancher instance (v2.7 or later)
- A managed Kubernetes cluster with a StorageClass configured for dynamic provisioning
- Access to a project and namespace in the cluster
- Familiarity with persistent volumes and claims

## Step 1: Prepare a StorageClass

StatefulSets typically require persistent storage. Verify that your cluster has a StorageClass available:

```bash
kubectl get storageclass
```

If no StorageClass is configured, you need to create one. For example, with a local-path provisioner:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

## Step 2: Navigate to the Workloads Page

In the Rancher dashboard, select your target cluster. From the left sidebar, click **Workloads** and then select **StatefulSets**.

## Step 3: Create a New StatefulSet

Click the **Create** button. Fill in the basic configuration:

- **Name**: Enter a name like `my-postgres`
- **Namespace**: Select your target namespace
- **Replicas**: Set the number of replicas (e.g., 3)
- **Service Name**: Enter a headless service name like `my-postgres-headless` (Rancher can create this for you)

## Step 4: Configure the Container

In the **Container** section:

- **Container Image**: Enter `postgres:15`
- **Pull Policy**: Select `IfNotPresent`

Add environment variables for PostgreSQL:

- Click **Add Variable**
- Add `POSTGRES_PASSWORD` with a value like `mysecretpassword`
- Add `POSTGRES_DB` with a value like `mydb`

Configure the container port:

- Click **Add Port**
- Set **Container Port** to `5432`
- Set **Protocol** to `TCP`

## Step 5: Add Volume Claim Templates

This is the key differentiator for StatefulSets. Scroll to the **Volume Claim Templates** section and click **Add Volume Claim Template**:

- **Name**: `postgres-data`
- **Storage Class**: Select your StorageClass
- **Access Mode**: `ReadWriteOnce`
- **Storage**: `10Gi`

Then mount the volume in your container:

- **Mount Point**: `/var/lib/postgresql/data`
- **Sub Path**: `postgres` (recommended for PostgreSQL)

## Step 6: Configure the Update Strategy

For StatefulSets, Rancher offers two update strategies:

- **RollingUpdate** (default): Updates pods in reverse ordinal order (from highest to lowest)
- **OnDelete**: Only updates pods when they are manually deleted

Set the **Partition** value if you want to perform a staged rollout. Pods with an ordinal greater than or equal to the partition value will be updated.

## Step 7: Deploy the StatefulSet

Click **Create** to deploy. Rancher will create the StatefulSet and its associated headless Service.

Watch the pods come up in order. StatefulSet pods are created sequentially (pod-0 first, then pod-1, then pod-2), and each pod must be Running and Ready before the next one is started.

## Alternative: Deploy via YAML

For more control, use the YAML editor:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-postgres-headless
  namespace: default
spec:
  clusterIP: None
  selector:
    app: my-postgres
  ports:
    - port: 5432
      targetPort: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-postgres
  namespace: default
spec:
  serviceName: my-postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: my-postgres
  template:
    metadata:
      labels:
        app: my-postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              value: mysecretpassword
            - name: POSTGRES_DB
              value: mydb
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
              subPath: postgres
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 500m
              memory: 1Gi
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: local-storage
        resources:
          requests:
            storage: 10Gi
```

## Step 8: Verify the StatefulSet

Check that all pods are running and have stable identities:

```bash
kubectl get statefulset my-postgres -n default
kubectl get pods -l app=my-postgres -n default
```

Each pod will have a predictable name: `my-postgres-0`, `my-postgres-1`, `my-postgres-2`.

Verify that PersistentVolumeClaims were created:

```bash
kubectl get pvc -l app=my-postgres -n default
```

## Accessing Individual Pods

One advantage of StatefulSets is DNS-based discovery. Each pod is addressable at:

```
<pod-name>.<headless-service>.<namespace>.svc.cluster.local
```

For example:

```
my-postgres-0.my-postgres-headless.default.svc.cluster.local
```

## Scaling a StatefulSet

To scale up or down:

1. Go to **Workloads > StatefulSets**
2. Click the three-dot menu and select **Edit Config**
3. Change the **Replicas** value
4. Click **Save**

When scaling down, pods are removed in reverse order (highest ordinal first). The associated PVCs are not deleted automatically, which preserves your data.

## Summary

You have successfully deployed a StatefulSet in Rancher. StatefulSets provide stable identities, ordered operations, and persistent storage for stateful applications. Rancher simplifies the management of StatefulSets through its UI while maintaining full access to the underlying Kubernetes primitives. Remember that StatefulSets are best suited for applications that genuinely need stable storage and identity; for stateless workloads, a Deployment is more appropriate.
