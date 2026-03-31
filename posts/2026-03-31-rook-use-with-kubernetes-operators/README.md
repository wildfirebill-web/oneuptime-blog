# How to Use Rook-Ceph with Kubernetes Operators

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Operator, Kubernetes, StorageClass, Automation

Description: Learn how to integrate Rook-Ceph storage with database and application operators on Kubernetes, enabling automated provisioning of persistent volumes for operator-managed workloads.

---

Kubernetes Operators automate the lifecycle of complex applications like databases and message queues. Rook-Ceph integrates with these operators by providing StorageClasses that operators use to provision persistent storage automatically when creating managed resources.

## How Operators Use Storage

Most database operators (PostgreSQL, MySQL, Kafka, etc.) accept a `storageClassName` in their CRD spec. The operator then creates PVCs using that StorageClass for the resources it manages. Rook-Ceph simply needs to provide appropriately configured StorageClasses.

## Creating Operator-Ready StorageClasses

Operators need different storage profiles depending on the workload:

```yaml
# High-performance block storage for databases
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-operator-db
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: database-pool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

## Integrating with CloudNativePG (PostgreSQL Operator)

CloudNativePG is a popular PostgreSQL operator that uses StorageClasses for data and WAL volumes:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster
  namespace: databases
spec:
  instances: 3
  storage:
    size: 100Gi
    storageClass: rook-ceph-operator-db
  walStorage:
    size: 20Gi
    storageClass: rook-ceph-operator-db
  backup:
    barmanObjectStore:
      destinationPath: s3://pg-backups
      endpointURL: http://rook-ceph-rgw-backup-store.rook-ceph.svc.cluster.local
      s3Credentials:
        accessKeyId:
          name: pg-backup-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: pg-backup-creds
          key: SECRET_ACCESS_KEY
```

## Integrating with Strimzi (Kafka Operator)

Strimzi deploys Apache Kafka and uses StorageClasses per broker:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-kafka
  namespace: messaging
spec:
  kafka:
    replicas: 3
    storage:
      type: persistent-claim
      size: 200Gi
      class: rook-ceph-operator-db
      deleteClaim: false
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      class: rook-ceph-operator-db
```

## Integrating with Redis Operator (Redis Enterprise)

Configure the Redis Operator to use Rook storage:

```yaml
apiVersion: app.redislabs.com/v1
kind: RedisEnterpriseCluster
metadata:
  name: rec
  namespace: redis
spec:
  nodes: 3
  redisEnterpriseNodeResources:
    requests:
      memory: 4Gi
  persistentSpec:
    enabled: true
    storageClassName: rook-ceph-operator-db
    volumeSize: 20Gi
```

## Monitoring PVC Lifecycle Across Operators

Track PVC usage across all operators:

```bash
# List all PVCs across namespaces with their StorageClass
kubectl get pvc -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,CLASS:.spec.storageClassName,SIZE:.spec.resources.requests.storage,STATUS:.status.phase'

# Find PVCs using Rook StorageClasses
kubectl get pvc -A | grep rook-ceph
```

## Summary

Rook-Ceph integrates with Kubernetes Operators by providing StorageClasses that operators reference when provisioning persistent volumes. CloudNativePG, Strimzi, and Redis operators all accept `storageClass` fields in their CRD specs. A well-designed set of Rook StorageClasses covering different performance profiles (with `reclaimPolicy: Retain` and `WaitForFirstConsumer`) gives operators the flexibility they need while protecting data from accidental deletion.
