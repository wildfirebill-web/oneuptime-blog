# How to Migrate Ceph from VM-Based to Container-Based Deployment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Migration, Kubernetes, Container

Description: Learn how to migrate a Ceph cluster running on bare VMs to a container-based deployment managed by Rook on Kubernetes.

---

Many organizations run Ceph directly on virtual machines. Migrating to Rook on Kubernetes provides automated lifecycle management, Kubernetes-native storage provisioning, and standardized operational practices.

## Assessing the Existing VM-Based Cluster

Document the current deployment:

```bash
ceph -s
ceph osd tree
ceph osd dump
ceph df
ceph mon dump
```

Record the Ceph version:

```bash
ceph version
```

## Migration Strategy

The safest migration path avoids moving data - instead, you adopt the existing cluster into Rook's management rather than creating a new cluster and migrating data.

For a full data migration (new cluster), use RGW multi-site and RBD mirroring as covered in datacenter migration guides.

## Adopting an Existing Cluster with Rook (Import)

Rook supports importing an existing Ceph cluster:

### Step 1 - Deploy Rook Operator

```bash
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/operator.yaml
```

### Step 2 - Create CephCluster with DataDirHostPath

Point Rook to the existing Ceph data directory:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v18.2.0
  dataDirHostPath: /var/lib/ceph
  mon:
    count: 3
    allowMultiplePerNode: false
  external:
    enable: false
  storage:
    useAllNodes: true
    useAllDevices: false
    devices:
    - name: sdb
```

### Step 3 - Match Existing Keyring

Copy existing keyrings to the expected location:

```bash
# On each node, ensure keyring is in place
cp /etc/ceph/ceph.client.admin.keyring /var/lib/rook/rook-ceph/
```

## Alternative: External Cluster Mode

Connect Rook to manage a pre-existing cluster without touching its daemons:

```bash
# Generate external cluster config
python3 create-external-cluster-resources.py \
  --rbd-data-pool-name=mypool \
  --namespace=rook-ceph-external \
  --format=bash | bash
```

This provides CSI storage classes backed by the existing cluster without containerizing the Ceph daemons.

## Containerizing Daemons Incrementally

Containerize one service type at a time:

1. Start with MGR (lowest risk)
2. Then add new MONs and remove old ones one by one
3. Finally, migrate OSDs using the add-new / remove-old process

```bash
# Add containerized OSD
kubectl apply -f new-osd-deployment.yaml

# Wait for cluster to rebalance
watch ceph -s

# Remove VM-based OSD
ceph osd out osd.3
ceph osd purge 3 --yes-i-really-mean-it
```

## Verifying the Migration

```bash
kubectl -n rook-ceph get pods
ceph -s
ceph health
kubectl get storageclass
kubectl get pv
```

## Summary

Migrating from VM-based to container-based Ceph can be done incrementally by importing the existing cluster into Rook using the shared data directory, or by using Rook's external cluster mode to gain Kubernetes storage class benefits without immediately containerizing daemons. The incremental daemon migration approach minimizes risk by containerizing one component at a time.
