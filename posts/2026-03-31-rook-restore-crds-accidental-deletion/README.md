# How to Restore CRDs After Accidental Deletion in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CRD, Kubernetes, Recovery

Description: Learn how to recover Rook-Ceph custom resource definitions after accidental deletion without losing cluster data or configuration.

---

Rook uses Kubernetes Custom Resource Definitions (CRDs) to define its storage objects: `CephCluster`, `CephBlockPool`, `CephFilesystem`, and many others. If these CRDs are accidentally deleted, Kubernetes loses awareness of the Rook resources, though the underlying Ceph cluster data typically survives on the disks.

## Understanding the Impact

When CRDs are deleted:
- All custom resources (CephCluster, CephBlockPool, etc.) are also deleted by Kubernetes
- The Rook operator loses its configuration and stops managing the cluster
- The underlying Ceph daemons may continue running for a short time
- StorageClasses referencing Rook provisioners stop working
- New PVC provisioning fails

Crucially, data stored on OSDs is not immediately destroyed. Recovery is possible.

## Identifying Deleted CRDs

Check which CRDs are missing:

```bash
kubectl get crd | grep rook
kubectl get crd | grep ceph
```

Compare against the expected list from the Rook documentation. A healthy installation includes:

```text
cephblockpools.ceph.rook.io
cephclusters.ceph.rook.io
cephfilesystems.ceph.rook.io
cephfilesystemmirrors.ceph.rook.io
cephfilesystemsubvolumegroups.ceph.rook.io
cephnfses.ceph.rook.io
cephobjectstores.ceph.rook.io
cephobjectstoreusers.ceph.rook.io
...
```

## Step 1: Reinstall CRDs

Reinstall the CRDs from the Rook release manifests that match your installed operator version:

```bash
ROOK_VERSION="v1.14.0"
kubectl apply -f https://raw.githubusercontent.com/rook/rook/${ROOK_VERSION}/deploy/examples/crds.yaml
```

If you installed Rook via Helm, reinstall the CRDs chart:

```bash
helm install rook-ceph-crds rook-release/rook-ceph-crds \
  --namespace rook-ceph \
  --version 1.14.0
```

Verify CRDs are restored:

```bash
kubectl get crd | grep ceph | wc -l
```

## Step 2: Restore Custom Resources from Backup

If you maintain backups of your Rook CRs (strongly recommended), apply them:

```bash
kubectl apply -f cephcluster-backup.yaml
kubectl apply -f cephblockpool-backup.yaml
kubectl apply -f cephfilesystem-backup.yaml
kubectl apply -f storageclasses-backup.yaml
```

If you do not have backups, you must recreate the CRs manually. The CephCluster CR must match the existing cluster configuration exactly - especially `dataDirHostPath`, `storage.nodes`, and device selectors.

## Step 3: Recreate CephCluster CR Manually

If no backup exists, examine the running Ceph daemons to reconstruct the configuration:

```bash
# Check which nodes have mon data
ls /var/lib/rook/

# Check OSD disk labels
lsblk -f | grep ceph
```

Create a minimal CephCluster CR that matches your existing setup:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v18.2.0
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: false
  storage:
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^sd[b-z]"
```

Apply this and allow the Rook operator to reconcile:

```bash
kubectl apply -f cephcluster-restored.yaml
```

## Step 4: Restart the Rook Operator

After restoring CRDs and CRs, restart the operator to trigger reconciliation:

```bash
kubectl -n rook-ceph rollout restart deployment rook-ceph-operator
kubectl -n rook-ceph rollout status deployment rook-ceph-operator
```

Watch the operator reconnect to the existing Ceph cluster:

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-operator -f | grep -E "reconcil|error|health"
```

## Step 5: Verify Recovery

Once the operator has reconciled, check cluster health:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
kubectl -n rook-ceph get cephcluster
```

StorageClasses should resume provisioning new PVCs. Existing PVs remain bound to their data since the underlying Ceph data was never deleted.

## Prevention

Prevent future accidental CRD deletion by adding a deletion protection annotation:

```bash
kubectl annotate crd cephclusters.ceph.rook.io \
  "helm.sh/resource-policy=keep"
```

Regular backups of CRDs and CRs using Velero or a simple cron job are essential for fast recovery.

## Summary

Restoring Rook CRDs after accidental deletion requires reinstalling the CRD manifests, recreating the custom resources from backups or by examining running daemons, and restarting the operator to trigger reconciliation. The underlying Ceph data survives on OSD disks through the process, making full recovery possible when the configuration can be accurately reconstructed.
