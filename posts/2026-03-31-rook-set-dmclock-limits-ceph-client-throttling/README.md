# How to Set DmClock Limits for Ceph Client Throttling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, QoS, DmClock, Throttling

Description: Learn how to set DmClock limits to cap the maximum I/O rate for Ceph clients, preventing noisy neighbors from overwhelming OSD resources.

---

## What are DmClock Limits?

A DmClock limit is the maximum I/O rate (ceiling) that a client can achieve on an OSD. Even if spare capacity exists, a client will not exceed its limit. This is the critical mechanism for preventing noisy-neighbor problems where a single client consumes all available OSD throughput.

## How Limits Differ from Reservations

| Parameter | Description | Effect |
|-----------|-------------|--------|
| Reservation | Minimum guaranteed rate | Client gets at least this |
| Weight | Proportional spare share | Determines ratio above floor |
| Limit | Maximum rate cap | Client never exceeds this |

Limits prevent over-consumption; reservations guarantee a minimum.

## Setting Global Client Limits

Set a ceiling for all client I/O globally:

```bash
# Allow max 2000 IOPS per OSD for client operations
ceph config set osd osd_mclock_scheduler_client_lim 2000

# Allow max 500 IOPS for background recovery
ceph config set osd osd_mclock_scheduler_background_recovery_lim 500

# Allow max 300 IOPS for best-effort operations
ceph config set osd osd_mclock_scheduler_background_best_effort_lim 300
```

## Per-OSD Limit Configuration

Configure tighter limits on slower HDDs to prevent latency spikes:

```bash
# HDD OSDs - strict limits
ceph config set osd.8 osd_mclock_scheduler_client_lim 300
ceph config set osd.9 osd_mclock_scheduler_client_lim 300
ceph config set osd.10 osd_mclock_scheduler_client_lim 300

# NVMe OSDs - higher limits
ceph config set osd.0 osd_mclock_scheduler_client_lim 8000
ceph config set osd.1 osd_mclock_scheduler_client_lim 8000
```

## RBD Image-Level Throttling

For finer-grained control, set I/O limits directly on RBD images:

```bash
# Limit a noisy VM's disk I/O
rbd config image set vms/vm-noisy rbd_qos_iops_limit 500
rbd config image set vms/vm-noisy rbd_qos_read_iops_limit 300
rbd config image set vms/vm-noisy rbd_qos_write_iops_limit 200

# Limit bandwidth in bytes per second
rbd config image set vms/vm-noisy rbd_qos_bps_limit 104857600   # 100MB/s

# Verify settings
rbd config image get vms/vm-noisy | grep qos
```

## Applying Kubernetes StorageClass QoS

In Kubernetes with Rook-Ceph, apply QoS via StorageClass parameters:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd-throttled
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
  # QoS limits
  qosIOPSLimit: "500"
  qosBPSLimit: "52428800"
```

## Testing Limit Enforcement

Verify limits are enforced under load:

```bash
# This should be throttled to ~500 IOPS for rbd_qos_iops_limit=500
fio --name=limit-test \
  --filename=/dev/rbd0 \
  --rw=randwrite \
  --bs=4k \
  --numjobs=8 \
  --iodepth=64 \
  --runtime=60 \
  --group_reporting | grep IOPS
```

The reported IOPS should not exceed the configured limit.

## Monitoring Throttled Clients

Identify which clients are being throttled:

```bash
ceph daemon osd.0 dump_mclock_queue
```

Check for queued/delayed operations:

```bash
ceph daemon osd.0 perf dump | python3 -m json.tool | grep -E "delayed|throttle"
```

## Summary

DmClock limits enforce a hard ceiling on the I/O rate any client or background process can achieve on Ceph OSDs. Setting limits per OSD type, globally via config, or per RBD image provides flexible throttling that protects shared OSD resources from noisy-neighbor effects. Combined with reservations, limits create a complete QoS envelope that guarantees both minimum and maximum I/O rates for each client.
