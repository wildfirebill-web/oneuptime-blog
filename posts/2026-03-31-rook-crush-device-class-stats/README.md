# How to View CRUSH Device Class Statistics in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CRUSH, Device Class, Storage

Description: View and analyze storage statistics broken down by CRUSH device class (HDD, SSD, NVMe) to understand utilization per hardware tier in Rook-Ceph.

---

Ceph CRUSH device classes allow you to group OSDs by hardware type (hdd, ssd, nvme) and create per-class storage pools. Monitoring statistics by device class helps you track utilization across different storage tiers.

## View Device Class Overview

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph df
```

The `RAW STORAGE` section breaks down by device class:

```text
--- RAW STORAGE ---
CLASS    SIZE     AVAIL    USED     RAW USED  %RAW USED
hdd      18 TiB   15 TiB   600 GiB  900 GiB   4.88
ssd      4 TiB    3.5 TiB  150 GiB  200 GiB   4.88
nvme     2 TiB    1.8 TiB   50 GiB   75 GiB    3.66
TOTAL    24 TiB   20.3 TiB 800 GiB  1.175 TiB  4.79
```

## List OSDs by Device Class

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd tree | grep -E "CLASS|hdd|ssd|nvme"
```

Full OSD tree with device class information:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd tree --format json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for node in data['nodes']:
    if node['type'] == 'osd':
        dc = node.get('device_class', 'unknown')
        print(f'osd.{node[\"id\"]}: class={dc}, status={node[\"status\"]}')
"
```

## View Per-Class OSD Usage

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd df tree --format json | python3 -c "
import json, sys
data = json.load(sys.stdin)
classes = {}
for node in data['nodes']:
    if node['type'] == 'osd':
        dc = node.get('device_class', 'unknown')
        if dc not in classes:
            classes[dc] = {'total': 0, 'used': 0, 'count': 0}
        classes[dc]['total'] += node.get('kb', 0)
        classes[dc]['used'] += node.get('kb_used', 0)
        classes[dc]['count'] += 1

for cls, stats in classes.items():
    total_gb = stats['total'] / (1024**2)
    used_gb = stats['used'] / (1024**2)
    pct = used_gb / total_gb * 100 if total_gb > 0 else 0
    print(f'{cls}: {stats[\"count\"]} OSDs, {used_gb:.1f}/{total_gb:.1f} GB ({pct:.1f}%)')
"
```

## List All Device Classes

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd crush class ls
```

## Get OSDs for a Specific Class

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd crush class ls-osd hdd

kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd crush class ls-osd ssd
```

## Set OSD Device Class

If Ceph auto-detected the wrong device class:

```bash
# Remove current class
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd crush rm-device-class osd.3

# Set correct class
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd crush set-device-class nvme osd.3
```

## Create a Class-Specific CRUSH Rule and Pool

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- bash -c "
  # Create rule targeting only SSD devices
  ceph osd crush rule create-replicated ssd-rule default host ssd

  # List rules
  ceph osd crush rule ls
"
```

In Rook, specify the device class in `CephBlockPool`:

```yaml
spec:
  failureDomain: host
  replicated:
    size: 3
  deviceClass: ssd
```

## Summary

CRUSH device class statistics in Ceph are visible in `ceph df` (raw storage by class), `ceph osd tree` (per-OSD class assignment), and `ceph osd crush class ls-osd <class>` (OSD membership per class). Use device class filtering in `CephBlockPool` to ensure workloads land on the correct hardware tier, and monitor per-class utilization to avoid filling one tier while another remains underutilized.
