# How to Configure Volume Snapshots in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Storage, Persistent Volumes

Description: A step-by-step guide to configuring and managing volume snapshots for persistent volumes in Rancher-managed Kubernetes clusters.

Volume snapshots provide point-in-time copies of your persistent volumes, enabling data backup, recovery, and cloning. Kubernetes supports volume snapshots through the CSI snapshot controller and compatible CSI drivers. This guide covers how to set up and use volume snapshots in Rancher.

## Prerequisites

- A running Rancher instance (v2.6 or later)
- A managed Kubernetes cluster
- A CSI driver that supports snapshots (EBS, Azure Disk, GCE PD, etc.)
- kubectl access to your cluster

## Step 1: Install the Snapshot Controller

Many clusters do not include the snapshot controller by default. Install it:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v7.0.1/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v7.0.1/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```

Install the snapshot CRDs:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v7.0.1/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v7.0.1/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v7.0.1/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
```

Verify:

```bash
kubectl get pods -n kube-system | grep snapshot
kubectl get crd | grep snapshot
```

## Step 2: Create a VolumeSnapshotClass

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapshot-class
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: ebs.csi.aws.com
deletionPolicy: Retain
```

For different storage backends:

```yaml
# Azure Disk

apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: azure-snapshot-class
driver: disk.csi.azure.com
deletionPolicy: Retain
parameters:
  incremental: "true"
---
# Google PD
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: gce-snapshot-class
driver: pd.csi.storage.gke.io
deletionPolicy: Retain
---
# vSphere
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: vsphere-snapshot-class
driver: csi.vsphere.vmware.com
deletionPolicy: Retain
```

```bash
kubectl apply -f snapshot-classes.yaml
```

## Step 3: Create a Volume Snapshot

Take a snapshot of an existing PVC:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-snapshot-20260319
  namespace: default
spec:
  volumeSnapshotClassName: csi-snapshot-class
  source:
    persistentVolumeClaimName: db-data
```

```bash
kubectl apply -f db-snapshot.yaml
```

## Step 4: Monitor Snapshot Status

```bash
# Check snapshot status
kubectl get volumesnapshot db-snapshot-20260319 -n default

# Get detailed information
kubectl describe volumesnapshot db-snapshot-20260319 -n default

# Check the snapshot content
kubectl get volumesnapshotcontent

# Verify readiness
kubectl get volumesnapshot db-snapshot-20260319 -o jsonpath='{.status.readyToUse}'
```

A successful snapshot shows `readyToUse: true`.

## Step 5: Restore from a Snapshot

Create a new PVC from the snapshot:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data-restored
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage
  resources:
    requests:
      storage: 50Gi
  dataSource:
    name: db-snapshot-20260319
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

```bash
kubectl apply -f restored-pvc.yaml
kubectl get pvc db-data-restored -n default
```

## Step 6: Clone a Volume Using Snapshots

Clone a volume by creating a snapshot and restoring it:

```bash
# Step 1: Create snapshot
kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: clone-source-snap
  namespace: default
spec:
  volumeSnapshotClassName: csi-snapshot-class
  source:
    persistentVolumeClaimName: production-data
EOF

# Step 2: Wait for snapshot
kubectl wait --for=jsonpath='{.status.readyToUse}'=true \
  volumesnapshot/clone-source-snap -n default --timeout=300s

# Step 3: Create clone PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: staging-data
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage
  resources:
    requests:
      storage: 50Gi
  dataSource:
    name: clone-source-snap
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
EOF
```

## Step 7: Create Scheduled Snapshots

Use a CronJob to create periodic snapshots:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scheduled-snapshot
  namespace: default
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-sa
          containers:
          - name: snapshot-creator
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              cat <<SNAP | kubectl apply -f -
              apiVersion: snapshot.storage.k8s.io/v1
              kind: VolumeSnapshot
              metadata:
                name: db-snap-${TIMESTAMP}
                namespace: default
              spec:
                volumeSnapshotClassName: csi-snapshot-class
                source:
                  persistentVolumeClaimName: db-data
              SNAP
          restartPolicy: OnFailure
```

Create the service account with required permissions:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: snapshot-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: snapshot-role
  namespace: default
rules:
- apiGroups: ["snapshot.storage.k8s.io"]
  resources: ["volumesnapshots"]
  verbs: ["create", "get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: snapshot-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount
  name: snapshot-sa
roleRef:
  kind: Role
  name: snapshot-role
  apiGroup: rbac.authorization.k8s.io
```

## Step 8: Manage Snapshot Retention

Clean up old snapshots to manage storage costs:

```bash
# List all snapshots sorted by creation time
kubectl get volumesnapshot -n default --sort-by='.metadata.creationTimestamp'

# Delete a specific snapshot
kubectl delete volumesnapshot db-snap-20260315-020000 -n default

# Delete snapshots older than 7 days
kubectl get volumesnapshot -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}' | \
  while read name timestamp; do
    age=$(( ($(date +%s) - $(date -d "$timestamp" +%s)) / 86400 ))
    if [ $age -gt 7 ]; then
      echo "Deleting snapshot $name (${age} days old)"
      kubectl delete volumesnapshot $name -n default
    fi
  done
```

## Step 9: Pre-Snapshot Hooks

Ensure data consistency before taking snapshots by quiescing the application:

```bash
# Freeze the filesystem before snapshot
kubectl exec db-pod -- fsfreeze --freeze /data

# Create snapshot
kubectl apply -f snapshot.yaml

# Wait for snapshot to be ready
kubectl wait --for=jsonpath='{.status.readyToUse}'=true volumesnapshot/db-snapshot --timeout=60s

# Unfreeze
kubectl exec db-pod -- fsfreeze --unfreeze /data
```

For databases, use native flush commands:

```bash
# MySQL - flush and lock
kubectl exec mysql-pod -- mysql -e "FLUSH TABLES WITH READ LOCK;"
# Take snapshot...
kubectl exec mysql-pod -- mysql -e "UNLOCK TABLES;"

# PostgreSQL - checkpoint
kubectl exec pg-pod -- psql -c "CHECKPOINT;"
# Take snapshot...
```

## Step 10: Troubleshoot Snapshot Issues

```bash
# Check snapshot controller logs
kubectl logs -n kube-system -l app=snapshot-controller --tail=50

# Check CSI driver logs
kubectl logs -n kube-system -l app=csi-controller --tail=50

# Verify VolumeSnapshotClass
kubectl get volumesnapshotclass

# Check VolumeSnapshotContent
kubectl get volumesnapshotcontent
kubectl describe volumesnapshotcontent <name>

# Common issues
# - Snapshot stuck: check CSI driver supports snapshots
# - readyToUse: false: check snapshot controller and CSI driver logs
# - Restore fails: verify snapshot class and storage class compatibility
```

## Summary

Volume snapshots in Rancher provide essential data protection for persistent volumes. By configuring the snapshot controller and VolumeSnapshotClasses, you can take point-in-time copies of your data, restore from backups, and clone volumes for testing environments. Combined with scheduled snapshots and retention policies, you can build a comprehensive data protection strategy for your Kubernetes workloads.
