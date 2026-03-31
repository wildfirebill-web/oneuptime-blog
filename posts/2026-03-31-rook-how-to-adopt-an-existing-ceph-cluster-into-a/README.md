# How to Adopt an Existing Ceph Cluster into Rook on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Migration, Adoption, Cluster Management

Description: Learn how to adopt an existing Ceph cluster managed outside of Kubernetes into Rook, enabling Kubernetes-native management without data loss.

---

## Overview

If you have an existing Ceph cluster that was deployed independently (using cephadm, ceph-deploy, or manually), you can bring it under Rook management. This is called cluster adoption or import, and it allows Kubernetes workloads to consume the existing Ceph cluster through the Rook CSI driver without migrating or re-creating the data.

## Prerequisites

Before adopting a Ceph cluster:
- The existing Ceph cluster must be healthy (`HEALTH_OK` or `HEALTH_WARN`)
- All monitors must be reachable from the Kubernetes cluster nodes
- You need the Ceph admin keyring from the existing cluster

## Step 1 - Gather Cluster Information

On the existing Ceph cluster, collect the required information:

```bash
# Get the cluster FSID
ceph fsid

# Get monitor addresses
ceph mon dump | grep "^[0-9]"

# Export the admin keyring
ceph auth get client.admin
```

Example output:

```text
FSID: abc12345-6789-...
Monitors:
  0: 192.168.1.10:6789/0 mon.a
  1: 192.168.1.11:6789/0 mon.b
  2: 192.168.1.12:6789/0 mon.c
```

## Step 2 - Create the Rook Namespace and Secrets

Create the namespace and necessary secrets in Kubernetes:

```bash
kubectl create namespace rook-ceph
```

Create the mon secret with the cluster keyring:

```bash
kubectl -n rook-ceph create secret generic rook-ceph-mon \
  --from-literal=ceph-username=client.admin \
  --from-literal=ceph-secret=<admin-key-from-existing-cluster>
```

Create the mon endpoint ConfigMap:

```bash
kubectl -n rook-ceph create configmap rook-ceph-mon-endpoints \
  --from-literal=data="a=192.168.1.10:6789\,b=192.168.1.11:6789\,c=192.168.1.12:6789" \
  --from-literal=mapping="" \
  --from-literal=maxMonId="2"
```

## Step 3 - Deploy the Rook Operator

Install the Rook operator without creating a new cluster:

```bash
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/common.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/operator.yaml
```

Wait for the operator to be ready:

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-operator
```

## Step 4 - Create the CephCluster with dataDirHostPath

Create a CephCluster CRD that references the existing cluster:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v18
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    count: 1
  dashboard:
    enabled: true
  storage:
    useAllNodes: false
    useAllDevices: false
  network:
    provider: host
```

Apply the cluster:

```bash
kubectl apply -f cephcluster.yaml
```

## Step 5 - Verify Adoption

Monitor the CephCluster status:

```bash
kubectl -n rook-ceph get cephcluster rook-ceph
```

Wait for the phase to become `Ready`:

```text
NAME        DATADIRHOSTPATH   MONCOUNT   AGE   PHASE   MESSAGE    HEALTH
rook-ceph   /var/lib/rook     3          5m    Ready   Cluster    HEALTH_OK
```

Verify connectivity through the toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

## Step 6 - Deploy CSI Drivers

Once the cluster is adopted, deploy the CSI drivers to enable dynamic provisioning:

```bash
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/csi/rbd/storageclass.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/csi/cephfs/storageclass.yaml
```

## Summary

Adopting an existing Ceph cluster into Rook involves gathering the cluster FSID, monitor addresses, and admin credentials, then creating the corresponding Kubernetes secrets and ConfigMaps before deploying the Rook operator and CephCluster CRD. Once adopted, the Rook operator manages the existing cluster and the CSI drivers enable Kubernetes workloads to consume existing pools without any data migration.
