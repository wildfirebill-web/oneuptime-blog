# How to Set OSD Device Classes in Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Storage, OSD, Device Class, Kubernetes

Description: Learn how to assign and use OSD device classes (HDD, SSD, NVMe) in Rook-Ceph to control data placement and optimize storage performance.

---

## What Are OSD Device Classes

Ceph automatically assigns a device class to each OSD based on the underlying hardware detected at runtime. The three standard device classes are `hdd`, `ssd`, and `nvme`. Device classes allow you to group OSDs by hardware type and create CRUSH rules that target specific classes, enabling tiered storage strategies within a single cluster.

When Rook deploys OSDs, it passes device type information to Ceph, which populates the CRUSH map accordingly. You can override the auto-detected class or explicitly assign classes in scenarios where detection is inaccurate or you want custom groupings.

## Viewing Current Device Classes

Use the Rook toolbox pod to inspect which class each OSD has been assigned:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
```

The output includes a `CLASS` column showing `hdd`, `ssd`, or `nvme` for each OSD. You can also query the CRUSH class list directly:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd crush class ls
```

## Overriding the Device Class for an OSD

If Ceph assigns the wrong class (common with virtual disks or certain NVMe drives that report as HDD), you can change it:

```bash
# Remove the existing class
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd crush rm-device-class osd.0

# Set the correct class
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd crush set-device-class ssd osd.0
```

Repeat for each OSD that needs reclassification. Device class changes take effect immediately in the CRUSH map.

## Setting Device Classes via the Rook CephCluster CRD

Rook does not expose a direct `deviceClass` field on individual OSD specs in all versions, but you can use the `devices` list under `storage.nodes` or `storage.storageClassDeviceSets` to hint at placement, then rely on CRUSH to distinguish classes. For PVC-based OSDs via `storageClassDeviceSets`, use separate sets per tier:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    storageClassDeviceSets:
      - name: ssd-set
        count: 3
        portable: true
        volumeClaimTemplates:
          - metadata:
              name: data
            spec:
              resources:
                requests:
                  storage: 500Gi
              storageClassName: fast-ssd
              volumeMode: Block
              accessModes:
                - ReadWriteOnce
      - name: hdd-set
        count: 6
        portable: true
        volumeClaimTemplates:
          - metadata:
              name: data
            spec:
              resources:
                requests:
                  storage: 4Ti
              storageClassName: slow-hdd
              volumeMode: Block
              accessModes:
                - ReadWriteOnce
```

Ceph will auto-detect the class from each PVC-backed device's performance characteristics, or you can set the class manually after deployment using the toolbox commands above.

## Creating CRUSH Rules for a Device Class

Once OSDs are classified, create a pool that targets a specific class using a replicated rule:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd crush rule create-replicated replicated-ssd default host ssd
```

Then apply that rule to a pool via the CephBlockPool CRD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ssd-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  deviceClass: ssd
  replicated:
    size: 3
```

The `deviceClass` field in the pool spec tells Rook to generate a CRUSH rule restricted to that class automatically.

## Practical Tiering Example

A common pattern is to store hot database volumes on SSD OSDs and cold archive data on HDD OSDs:

```bash
# Check OSD class distribution
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd df tree | grep -E "osd|CLASS"
```

Use separate StorageClasses pointing to separate CephBlockPools with different `deviceClass` values to route workloads to the appropriate tier.

## Summary

OSD device classes in Rook-Ceph let you organize storage hardware into logical tiers and build CRUSH rules that target specific classes. You can rely on auto-detection for most hardware, override incorrect assignments using the toolbox, and reference device classes directly in CephBlockPool and CephFilesystem pool specs. This gives fine-grained control over where data lands without manual CRUSH map editing.
