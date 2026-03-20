# How to Configure Longhorn CSI Snapshotter

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, CSI, Snapshots, Kubernetes, VolumeSnapshot, Storage, SUSE Rancher

Description: Learn how to configure the Longhorn CSI snapshotter to create, manage, and restore Kubernetes VolumeSnapshots using the standard CSI snapshot API.

---

Longhorn integrates with the Kubernetes CSI snapshots API, allowing you to create, list, and restore volume snapshots using standard Kubernetes objects (`VolumeSnapshot`, `VolumeSnapshotClass`, `VolumeSnapshotContent`).

---

## Step 1: Install CSI Snapshot Controller

The CSI snapshot controller must be installed in the cluster before Longhorn can use the snapshot API:

```bash
# Install the CSI snapshot CRDs
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml

# Install the snapshot controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```

---

## Step 2: Create a VolumeSnapshotClass

```yaml
# longhorn-snapshotclass.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: longhorn-snapshot-vsc
  annotations:
    # Make this the default snapshot class
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: driver.longhorn.io
deletionPolicy: Delete
parameters:
  # Use "true" to create a backup (backed up to S3/NFS)
  # Use "false" for a local Longhorn snapshot only
  type: snap
```

---

## Step 3: Create a VolumeSnapshot

```yaml
# volume-snapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: myapp-data-snapshot-v1
  namespace: my-app
spec:
  volumeSnapshotClassName: longhorn-snapshot-vsc
  source:
    # Reference the PVC to snapshot
    persistentVolumeClaimName: myapp-data
```

```bash
kubectl apply -f volume-snapshot.yaml

# Check snapshot is ready
kubectl get volumesnapshot myapp-data-snapshot-v1 -n my-app
```

---

## Step 4: Restore from a VolumeSnapshot

Create a new PVC from the snapshot:

```yaml
# restored-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data-restored
  namespace: my-app
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
  # Restore from the snapshot
  dataSource:
    name: myapp-data-snapshot-v1
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

---

## Step 5: List and Delete Snapshots

```bash
# List all snapshots
kubectl get volumesnapshot -A

# List snapshot contents (the underlying Longhorn resources)
kubectl get volumesnapshotcontent

# Delete a snapshot
kubectl delete volumesnapshot myapp-data-snapshot-v1 -n my-app
```

---

## Step 6: Automate Snapshots via Longhorn Recurring Jobs

```yaml
# recurring-snapshot-job.yaml
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: daily-snapshot
  namespace: longhorn-system
spec:
  cron: "0 2 * * *"   # Daily at 2 AM
  task: snapshot
  groups:
    - default
  retain: 7            # Keep last 7 daily snapshots
  concurrency: 2
```

---

## Best Practices

- Use `type: backup` in the VolumeSnapshotClass to create off-cluster backups (requires a Longhorn backup target).
- Schedule CSI snapshots via Longhorn RecurringJobs for consistent, automated protection.
- Test snapshot restore regularly — create a restore and verify data integrity before relying on snapshots for disaster recovery.
