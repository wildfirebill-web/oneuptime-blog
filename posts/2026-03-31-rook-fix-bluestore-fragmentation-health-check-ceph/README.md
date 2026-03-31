# How to Fix BLUESTORE_FRAGMENTATION Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, BlueStore, Fragmentation, Performance

Description: Learn how to diagnose and resolve the BLUESTORE_FRAGMENTATION health warning in Ceph when BlueStore block device allocation becomes heavily fragmented.

---

## Understanding BLUESTORE_FRAGMENTATION

BlueStore manages its own block allocator directly on top of raw devices. Over time, as objects are written and deleted, the free space on a BlueStore OSD can become fragmented - split into many small non-contiguous extents rather than a few large ones. `BLUESTORE_FRAGMENTATION` fires when the fragmentation score exceeds the warning threshold (default 0.7 on a scale of 0 to 1).

Check current health:

```bash
ceph health detail
```

Example output:

```text
HEALTH_WARN bluestore fragmentation is high
[WRN] BLUESTORE_FRAGMENTATION: osd.4 fragmentation score is 0.83
```

## Measuring Fragmentation Score

Check the current fragmentation score for each OSD:

```bash
ceph daemon osd.4 bluestore stats 2>/dev/null | grep -i frag
```

Or via Prometheus:

```bash
curl -s http://localhost:9283/metrics | grep bluestore_frag
```

In Rook:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  ceph tell osd.* bluestore stats 2>/dev/null | grep fragmentation
```

## Understanding the Fragmentation Score

The score ranges from 0.0 (no fragmentation) to 1.0 (completely fragmented):
- `< 0.7`: Normal, no action needed
- `0.7 - 0.9`: Warning - consider defragmentation
- `> 0.9`: Critical - performance is significantly degraded

Fragmentation causes slower write performance because BlueStore must scatter writes across many small free extents instead of one contiguous region.

## Fix: Defragment by Rebalancing OSD Data

The primary way to defragment BlueStore is to temporarily reduce the OSD's CRUSH weight to migrate data off, then restore the weight:

```bash
# Reduce weight to migrate data off (triggers rebalancing away)
ceph osd reweight 4 0.0

# Wait for data to fully migrate off
ceph -w
# Wait until all PGs are active+clean

# Restore weight (data migrates back, landing contiguously)
ceph osd reweight 4 1.0
```

When data is written back to the OSD, it lands in contiguous free space, effectively defragmenting the disk.

## Configuring Fragmentation Thresholds

Adjust when the warning fires:

```bash
# Warning threshold (default 0.7)
ceph config set osd bluestore_fragmentation_score_warn 0.75

# Critical threshold (default 0.9)
ceph config set osd bluestore_fragmentation_score_alert 0.92
```

## Enabling BlueStore Compression

Compression can reduce fragmentation by writing smaller, more uniformly-sized objects:

```bash
ceph config set osd bluestore_compression_mode aggressive
ceph config set osd bluestore_compression_algorithm zstd
```

Note: Compression helps prevent future fragmentation but does not defrag existing data.

## Monitoring Fragmentation Trends

Track fragmentation over time with Prometheus:

```yaml
- alert: BlueStoreFragmentation
  expr: ceph_bluestore_fragmentation_score > 0.7
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "OSD {{ $labels.ceph_daemon }} fragmentation is {{ $value }}"
    description: "Consider defragmenting by reweighting the OSD to 0 and back."
```

## Automatic Defragmentation

Ceph does not have automatic defragmentation. The reweight approach is the primary tool. To automate it:

```bash
#!/bin/bash
# Find OSDs with fragmentation > 0.8 and reweight them
ceph osd df -f json | python3 -c "
import sys, json
data = json.load(sys.stdin)
for node in data.get('nodes', []):
    if node.get('type') == 'osd' and node.get('fragmentation_score', 0) > 0.8:
        print(f'Defragging osd.{node[\"id\"]}')
"
```

## Summary

`BLUESTORE_FRAGMENTATION` warns that a BlueStore OSD's block allocator has a high fragmentation score, leading to degraded write performance. The primary fix is to temporarily reweight the OSD to 0.0 to migrate data off, then restore the weight so data writes back contiguously. Adjust thresholds to tune when warnings fire, and enable compression to reduce future fragmentation from varied object sizes.
