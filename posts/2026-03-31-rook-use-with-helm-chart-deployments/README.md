# How to Use Rook-Ceph with Helm Chart Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Helm, Kubernetes, Deployment, Storage

Description: Learn how to deploy and configure Rook-Ceph using Helm charts, including operator installation, cluster creation, and integrating Ceph storage with application Helm releases.

---

Helm is the de facto package manager for Kubernetes. Rook provides official Helm charts that simplify deploying the operator and creating Ceph clusters. Understanding how to integrate Rook-Ceph with application Helm deployments makes storage provisioning a repeatable, version-controlled process.

## Installing the Rook Operator with Helm

Add the Rook Helm repository and install the operator:

```bash
helm repo add rook-release https://charts.rook.io/release
helm repo update

# Install the Rook-Ceph operator
helm install --create-namespace \
  --namespace rook-ceph \
  rook-ceph rook-release/rook-ceph \
  --version v1.15.0
```

Verify the operator is running:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-operator
```

## Installing the Rook-Ceph Cluster with Helm

The cluster chart creates the CephCluster resource and associated components:

```bash
helm install --create-namespace \
  --namespace rook-ceph \
  rook-ceph-cluster rook-release/rook-ceph-cluster \
  --set operatorNamespace=rook-ceph
```

Customize the cluster values:

```yaml
# values.yaml for rook-ceph-cluster
cephClusterSpec:
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: false
  storage:
    useAllNodes: true
    useAllDevices: true
  resources:
    mgr:
      requests:
        cpu: 500m
        memory: 512Mi

cephBlockPools:
  - name: replicapool
    spec:
      replicated:
        size: 3
    storageClass:
      enabled: true
      name: rook-ceph-block
      isDefault: true
      reclaimPolicy: Delete

cephObjectStores:
  - name: my-store
    spec:
      gateway:
        instances: 2
    storageClass:
      enabled: true
      name: rook-ceph-bucket
```

Install with the custom values:

```bash
helm install rook-ceph-cluster rook-release/rook-ceph-cluster \
  -f values.yaml \
  --namespace rook-ceph
```

## Integrating Ceph Storage in Application Helm Charts

Reference the Rook-provisioned StorageClass in your application's Helm values:

```yaml
# values.yaml for a database chart
persistence:
  enabled: true
  storageClass: rook-ceph-block
  size: 20Gi
  accessMode: ReadWriteOnce
```

For applications with Helm sub-charts:

```yaml
# Parent chart values.yaml
postgresql:
  primary:
    persistence:
      storageClass: rook-ceph-block
      size: 50Gi
```

## Upgrading Rook with Helm

Keep the operator up to date:

```bash
helm repo update

# Check for new versions
helm search repo rook-release/rook-ceph --versions | head -5

# Upgrade the operator
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  --version v1.16.0
```

## Storing Helm Values in GitOps

For reproducible deployments, store Helm values in git:

```bash
# Directory structure for GitOps
infrastructure/
  rook/
    operator-values.yaml
    cluster-values.yaml
    kustomization.yaml
```

```yaml
# kustomization.yaml
helmCharts:
  - name: rook-ceph
    repo: https://charts.rook.io/release
    version: v1.15.0
    releaseName: rook-ceph
    namespace: rook-ceph
    valuesFile: operator-values.yaml
```

## Summary

Rook provides official Helm charts for both the operator and the cluster, making installation and upgrades declarative and version-controlled. Application Helm charts integrate with Rook storage by referencing the StorageClass name in their persistence values. Storing Helm values in a GitOps repository ensures reproducible cluster configurations and simplifies disaster recovery procedures.
