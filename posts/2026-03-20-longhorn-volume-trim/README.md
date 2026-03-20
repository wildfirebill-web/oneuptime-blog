# How to Configure Longhorn Volume Trim - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Volume Trim, Trim, Storage Optimization, Kubernetes, Disk Space, SUSE Rancher

Description: Learn how to configure and use Longhorn volume trim to reclaim unused disk space by discard trimming file system blocks that have been freed, reducing storage costs.

---

When you delete files from a Longhorn volume, the underlying disk space is not immediately reclaimed. Longhorn's volume trim (discard) feature propagates TRIM/UNMAP operations to the storage backend, recovering the unused space.

---

## How Volume Trim Works

When a file is deleted inside a Longhorn volume, the filesystem marks those blocks as free. Without TRIM, Longhorn continues to replicate those "freed" blocks. With TRIM enabled, Longhorn receives discard commands and deallocates the underlying disk blocks, reducing actual storage usage.

---

## Step 1: Enable Filesystem Trim in Your Workload

The filesystem inside the volume must be formatted with trim support and mounted with the `discard` option:

```yaml
# pod-with-trim.yaml

spec:
  containers:
    - name: app
      image: ubuntu:22.04
      command:
        - sh
        - -c
        - |
          # Format with discard support and mount
          # Note: this is for illustration - use initContainers for setup
          mount | grep /data
          fstrim /data   # manually trigger TRIM
      volumeMounts:
        - name: data
          mountPath: /data
```

For ext4 or xfs volumes, use `discard` mount option in the pod or use periodic `fstrim`:

```yaml
# initContainer to enable periodic fstrim
initContainers:
  - name: setup-trim
    image: ubuntu:22.04
    command: ["sh", "-c", "echo '0 * * * * fstrim /data' | crontab -"]
    securityContext:
      privileged: true
    volumeMounts:
      - name: data
        mountPath: /data
```

---

## Step 2: Trigger Volume Trim via Longhorn UI

In the Longhorn UI:

1. Navigate to **Volumes**
2. Select the volume you want to trim
3. Click **Trim Filesystem**

---

## Step 3: Trigger Volume Trim via Longhorn API

```bash
# Get the volume name
kubectl get lhvolume -n longhorn-system | grep my-pvc

# Trigger trim via the Longhorn API
LONGHORN_URL=http://longhorn-frontend.longhorn-system.svc.cluster.local

curl -X POST \
  ${LONGHORN_URL}/v1/volumes/<volume-name>?action=trimFilesystem \
  -H "Content-Type: application/json"
```

---

## Step 4: Enable Automatic Trim via StorageClass

Configure volumes to automatically enable the discard mount option:

```yaml
# storageclass-with-trim.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-trimmed
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  # Allow discard operations from the guest filesystem
  disableRevisionCounter: "false"
  unmapMarkSnapChainRemoved: "enabled"
```

---

## Step 5: Monitor Space Reclaimed

```bash
# Before trim - check allocated size
kubectl exec -n longhorn-system \
  longhorn-manager-xxxxx -- \
  longhorn-manager --volume-name <name> space-info

# After trim - compare actual vs allocated size
kubectl get lhvolume <name> -n longhorn-system \
  -o jsonpath='{.status.actualSize}'
```

---

## Best Practices

- Run `fstrim` on a schedule (daily or weekly) rather than continuously using the `discard` mount option - continuous discard has higher I/O overhead.
- Trim is most impactful for volumes with high file churn (log volumes, cache directories).
- After trimming, re-run Longhorn backups to capture the smaller effective data size.
