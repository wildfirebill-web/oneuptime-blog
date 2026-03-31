# How to Use Flat and Positional Weight Set Modes in CRUSH

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CRUSH, Weight Set, Storage

Description: Understand the difference between flat and positional CRUSH weight set modes in Ceph and when to use each for optimal data distribution balancing.

---

## Weight Set Modes in Ceph CRUSH

Ceph CRUSH weight sets support two internal modes for how weights are stored and applied across the hierarchy:

- **Flat mode** - stores a single weight per OSD, applied uniformly regardless of position in the CRUSH tree
- **Positional mode** - stores separate weights for each OSD at each position in the replica selection, allowing replica-specific tuning

Understanding these modes helps when inspecting balancer output and when diagnosing why specific OSDs may be over- or under-utilized in particular positions.

## How Flat Weight Sets Work

In flat mode, each OSD has one weight value in the weight set. This weight applies every time that OSD is considered, regardless of which replica is being placed:

```text
choose_args {
    {
        bucket_id -2  # host node-01
        weight_set [
            [ 1.1 0.9 ]  # weights for osd.0 and osd.1 (flat: one row)
        ]
    }
}
```

Flat mode is simpler and requires less memory. It is appropriate for most clusters where a uniform adjustment per OSD is sufficient to correct imbalances.

## How Positional Weight Sets Work

In positional mode, the weight set contains multiple rows - one per replica position. This allows different weights for the same OSD depending on whether it is being selected as the first, second, or third replica:

```text
choose_args {
    {
        bucket_id -2  # host node-01
        weight_set [
            [ 1.2 0.8 ]   # position 0 (primary replica) weights
            [ 0.9 1.1 ]   # position 1 (secondary replica) weights
            [ 1.0 1.0 ]   # position 2 (tertiary replica) weights
        ]
    }
}
```

This is used by the balancer's `crush-compat` mode when it detects that position-specific adjustments produce better results than flat adjustments alone.

## Checking Which Mode is in Use

```bash
# Export and inspect the CRUSH map
ceph osd getcrushmap -o crush.bin
crushtool -d crush.bin -o crush.txt
grep -A20 "choose_args" crush.txt

# Count rows per bucket in weight_set:
# 1 row = flat mode
# Multiple rows = positional mode

# Check balancer mode
ceph balancer status | grep mode
```

## Balancer Mode and Weight Set Interaction

```bash
# crush-compat mode uses compat weight sets (flat or positional)
ceph balancer mode crush-compat

# upmap mode uses per-PG upmap entries (different mechanism entirely)
ceph balancer mode upmap

# Show which weight sets exist
ceph osd crush dump | python3 -m json.tool | grep -c "weight_set"
```

## Forcing Flat Weight Sets

If you want to ensure flat mode is used (simpler, less memory):

```bash
# Disable automatic balancer
ceph balancer off

# Clear existing weight sets
ceph osd crush weight-set rm *

# Re-enable with crush-compat mode
ceph balancer mode crush-compat
ceph balancer on

# The balancer will start with flat weight sets
# and only move to positional if needed
```

## Inspecting Weight Set Rows for Debugging

```bash
# Show detailed CRUSH dump including all weight set rows
ceph osd crush dump | python3 -m json.tool > /tmp/crush-dump.json

# Count weight set rows per bucket to determine mode
python3 <<'EOF'
import json
with open('/tmp/crush-dump.json') as f:
    crush = json.load(f)
for args in crush.get('choose_args', {}).values():
    for bucket in args:
        rows = len(bucket.get('weight_set', []))
        print(f"bucket {bucket['bucket_id']}: {rows} weight set rows")
EOF
```

## Summary

Flat weight sets store one weight per OSD and apply universally, while positional weight sets store per-replica-position weights for finer tuning. The Ceph balancer in `crush-compat` mode manages both automatically, starting flat and adding positional rows when needed. In most cases you do not need to manually choose between modes - the balancer selects the appropriate format during optimization. Understanding the difference helps when reading CRUSH map output and diagnosing residual distribution imbalances.
