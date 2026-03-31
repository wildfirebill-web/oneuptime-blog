# How to Configure Bandwidth Limits for Ceph Clients

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, QoS, Bandwidth, Throttling

Description: Learn how to configure read and write bandwidth limits for Ceph clients at the RBD image, pool, and OSD levels to prevent bandwidth saturation.

---

## Bandwidth Limiting in Ceph

Bandwidth limits (bytes per second) complement IOPS limits by capping the throughput of large sequential I/O operations. While IOPS limits protect against many small random I/Os, bandwidth limits are essential for preventing large file copies or streaming workloads from saturating network or disk bandwidth.

## RBD Image Bandwidth Limits

Set bandwidth limits on RBD images in bytes per second:

```bash
# 200 MB/s read limit
rbd config image set mypool/large-vm rbd_qos_read_bps_limit 209715200

# 100 MB/s write limit
rbd config image set mypool/large-vm rbd_qos_write_bps_limit 104857600

# 250 MB/s total bandwidth limit
rbd config image set mypool/large-vm rbd_qos_bps_limit 262144000
```

Add burst bandwidth for short peaks:

```bash
# Allow 500 MB/s burst for up to 10 seconds
rbd config image set mypool/large-vm rbd_qos_bps_burst 524288000
rbd config image set mypool/large-vm rbd_qos_bps_burst_seconds 10
```

Verify the settings:

```bash
rbd config image list mypool/large-vm | grep bps
```

## Pool-Level Bandwidth Defaults

Apply bandwidth defaults to all new images in a pool using pool config:

```bash
rbd config pool set mypool rbd_qos_bps_limit 104857600        # 100 MB/s default
rbd config pool set mypool rbd_qos_read_bps_limit 78643200    # 75 MB/s read default
rbd config pool set mypool rbd_qos_write_bps_limit 52428800   # 50 MB/s write default
```

Individual image settings override pool defaults.

## Global Client Bandwidth Limits

Apply a global bandwidth limit that affects all RBD clients:

```bash
ceph config set client rbd_qos_bps_limit 209715200  # 200 MB/s global
```

This is useful as a safety net to prevent any single client from consuming all available bandwidth.

## Kubernetes PVC Bandwidth Limits

In Rook-Ceph, apply bandwidth limits at the StorageClass level:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rbd-bandwidth-limited
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
  mapOptions: "qos_bps_limit=104857600"
reclaimPolicy: Delete
```

## Network-Level Bandwidth Control

For multi-tenant environments, also apply Linux traffic control (tc) on the storage network:

```bash
# Limit a tenant's VM to 1Gbit/s on storage interface
tc qdisc add dev eth0 root handle 1: htb default 30
tc class add dev eth0 parent 1: classid 1:1 htb rate 10gbit
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 1gbit
tc filter add dev eth0 parent 1:0 protocol ip u32 \
  match ip src 10.0.1.50/32 flowid 1:10
```

## Verifying Bandwidth Limits

Test that bandwidth limits are enforced:

```bash
# Sequential write test - should plateau at the limit
fio --name=bw-test \
  --filename=/dev/rbd0 \
  --rw=write \
  --bs=1M \
  --numjobs=2 \
  --iodepth=8 \
  --runtime=60 \
  --group_reporting | grep -E "bw=|BW"
```

Monitor throughput in real time:

```bash
rbd perf image iostat --pool mypool -p 2
```

## Summary

Configuring bandwidth limits for Ceph clients protects shared storage networks from saturation caused by large sequential workloads. Setting limits at the RBD image level provides the most granular control, while pool-level defaults and global config options apply broader constraints. Combining bandwidth limits with burst allowances ensures applications get the throughput they need for short peaks while adhering to sustained limits.
