# How to Configure osd_recovery_sleep for Throttling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Recovery, Throttling, Configuration, Storage

Description: Learn how to configure osd_recovery_sleep in Ceph to throttle recovery operations by media type, reducing impact on client I/O during OSD healing.

---

## What Is osd_recovery_sleep?

`osd_recovery_sleep` introduces a sleep delay between OSD recovery operations. By pausing briefly between each recovery step, the OSD has time to process client I/O requests, reducing the recovery-induced latency spike for applications.

Ceph provides media-type-specific variants for more precise control:
- `osd_recovery_sleep` - global setting for all media
- `osd_recovery_sleep_hdd` - for spinning hard drives
- `osd_recovery_sleep_ssd` - for solid-state drives
- `osd_recovery_sleep_hybrid` - for mixed HDD/SSD configurations

## Default Values

| Parameter | Default |
|-----------|---------|
| `osd_recovery_sleep` | 0 |
| `osd_recovery_sleep_hdd` | 0.1 |
| `osd_recovery_sleep_ssd` | 0 |
| `osd_recovery_sleep_hybrid` | 0.025 |

HDDs have a non-zero default because their mechanical nature makes them more sensitive to concurrent recovery and client I/O.

## Checking Current Settings

```bash
ceph config get osd osd_recovery_sleep_hdd
ceph config get osd osd_recovery_sleep_ssd
ceph config dump | grep recovery_sleep
```

## Setting Recovery Sleep

### For spinning disk clusters (HDDs)

```bash
ceph config set osd osd_recovery_sleep_hdd 0.5
```

### For all-NVMe clusters

```bash
ceph config set osd osd_recovery_sleep_ssd 0.0
```

### For hybrid clusters

```bash
ceph config set osd osd_recovery_sleep_hybrid 0.1
```

### Apply immediately via injectargs

```bash
ceph tell osd.* injectargs '--osd-recovery-sleep-hdd=0.5'
```

## Tuning Guidelines

Use these starting points based on workload type:

| Workload Type | HDD Sleep | SSD Sleep |
|---------------|-----------|-----------|
| Low latency production | 0.5s | 0.02s |
| High throughput batch | 0.1s | 0.0s |
| Maintenance window | 0.0s | 0.0s |

Measure the impact on client latency after changing values:

```bash
ceph osd perf
rados bench -p <pool-name> 10 write --no-cleanup
```

## Rook CephCluster Configuration

Configure recovery sleep in the CephCluster spec for persistent settings:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephConfig:
    osd:
      osd_recovery_sleep_hdd: "0.5"
      osd_recovery_sleep_ssd: "0.0"
      osd_recovery_sleep_hybrid: "0.1"
```

Apply via toolbox for immediate effect:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_recovery_sleep_hdd 0.5
```

## Balancing Sleep and Recovery Time

Longer sleep values protect client latency but extend recovery time. Monitor both:

```bash
watch -n 5 ceph -s
```

If recovery is taking too long and the cluster remains at risk, temporarily reduce sleep during off-peak hours:

```bash
ceph config set osd osd_recovery_sleep_hdd 0.1
```

## Summary

`osd_recovery_sleep` is a powerful throttling lever that introduces pauses between recovery operations to protect client I/O. Media-specific variants allow fine-tuned control for HDD, SSD, and hybrid clusters. In Rook, these settings are managed through the CephCluster CRD, with runtime adjustments available via the toolbox pod.
