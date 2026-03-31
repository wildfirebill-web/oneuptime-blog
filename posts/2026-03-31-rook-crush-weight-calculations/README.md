# How to Understand CRUSH Map Weight Calculations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CRUSH, Weight, Storage

Description: Understand how Ceph CRUSH map weights are calculated, how bucket weights aggregate OSD weights, and how they influence data distribution across the cluster.

---

## How CRUSH Weights Work

In Ceph, every device (OSD) and bucket (host, rack, root) in the CRUSH map has a weight. OSD weights represent the device's relative storage capacity, while bucket weights are automatically calculated as the sum of their children's weights. The CRUSH algorithm uses these weights to determine the probability that each OSD is selected for data placement - a larger weight means more data is directed to that OSD.

## OSD Weight Convention

OSD weights are conventionally set equal to the OSD's capacity in terabytes:

```text
1 TB drive  -> CRUSH weight = 1.0
2 TB drive  -> CRUSH weight = 2.0
4 TB drive  -> CRUSH weight = 4.0
10 TB drive -> CRUSH weight = 10.0
```

This ensures that larger drives receive proportionally more PGs and data.

## Viewing Current Weights

```bash
# View weights for all OSDs and buckets
ceph osd tree

# Show detailed utilization with weights
ceph osd df tree

# Check a specific OSD's CRUSH weight
ceph osd tree -f json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for node in data['nodes']:
    if node.get('name') == 'osd.0':
        print('crush_weight:', node['crush_weight'])
        print('reweight:', node['reweight'])
"
```

## Bucket Weight Aggregation

Bucket weights are the sum of all child weights:

```text
host node-01:
  osd.0 weight=1.0
  osd.1 weight=2.0
  osd.2 weight=1.0
  -> host weight = 4.0

rack rack1:
  host node-01 weight=4.0
  host node-02 weight=3.0
  -> rack weight = 7.0

root default:
  rack rack1 weight=7.0
  rack rack2 weight=8.0
  -> root weight = 15.0
```

Ceph recalculates bucket weights automatically when OSD weights change.

## Weight vs Reweight

```bash
# CRUSH weight: structural capacity-based weight in the CRUSH map
ceph osd crush reweight osd.0 2.0

# Reweight: dynamic multiplier (0.0-1.0) applied at placement time
ceph osd reweight osd.0 0.8

# Effective weight = crush_weight * reweight
# osd.0: 2.0 * 0.8 = 1.6 effective weight
```

## Calculating Expected Data Per OSD

With uniform weights, each OSD receives data proportional to its weight:

```bash
# Formula: expected_fraction = osd_weight / root_weight
# Example: osd.0 weight=2.0, root weight=15.0
# Expected fraction = 2.0 / 15.0 = 13.3%

# Check actual vs expected with osd df
ceph osd df | awk 'NR==1 || NR>1 {printf "OSD %s: weight=%.3f, util=%s%%, var=%s\n", $1, $3, $9, $10}'
```

The `VAR` column in `ceph osd df` shows actual vs expected ratio. Values near 1.0 indicate good balance.

## Rebalancing After Weight Changes

Changing CRUSH weights triggers data rebalancing:

```bash
# Check how much data will move after a weight change
# First, note current mapping for a set of PGs
ceph pg dump pgs | awk '{print $1, $14}' | head -20 > before.txt

# Apply weight change
ceph osd crush reweight osd.0 2.0

# Wait for rebalancing, then compare
ceph pg dump pgs | awk '{print $1, $14}' | head -20 > after.txt
diff before.txt after.txt | grep "^>" | wc -l
echo "PGs remapped"
```

## Gradually Adjusting Weights

To avoid rebalancing storms, gradually change weights:

```bash
#!/bin/bash
OSD=$1
TARGET_WEIGHT=$2
CURRENT=$(ceph osd tree -f json | python3 -c "
import json, sys; d=json.load(sys.stdin)
[print(n['crush_weight']) for n in d['nodes'] if n.get('name')=='osd.$OSD']
")

STEPS=5
for i in $(seq 1 $STEPS); do
  NEW=$(python3 -c "print($CURRENT + ($TARGET_WEIGHT - $CURRENT) * $i / $STEPS)")
  ceph osd crush reweight osd.$OSD $NEW
  echo "Set osd.$OSD weight to $NEW, waiting..."
  sleep 30
done
```

## Summary

CRUSH weights determine data distribution in Ceph by representing OSD capacity (conventionally in TB). Bucket weights aggregate their children's weights automatically. The `VAR` column in `ceph osd df` indicates balance quality - values close to 1.0 mean the OSD carries its expected share. Use `ceph osd crush reweight` for permanent capacity-based adjustments and `ceph osd reweight` for temporary fine-tuning, and change weights gradually in production to avoid rebalancing storms.
