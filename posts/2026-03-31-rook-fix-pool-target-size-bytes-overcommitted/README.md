# How to Fix POOL_TARGET_SIZE_BYTES_OVERCOMMITTED Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Pool, Health Check, Capacity

Description: Learn how to fix POOL_TARGET_SIZE_BYTES_OVERCOMMITTED in Ceph, a warning that the sum of pool target sizes exceeds total available cluster capacity.

---

## What Is POOL_TARGET_SIZE_BYTES_OVERCOMMITTED?

`POOL_TARGET_SIZE_BYTES_OVERCOMMITTED` is a Ceph health warning that occurs when the sum of `target_size_bytes` values configured across all pools exceeds the total raw capacity of the cluster. The `target_size_bytes` setting is an optional hint to the PG autoscaler about how large each pool is expected to grow.

When pools are over-committed in terms of target sizes, the autoscaler's PG calculations become inaccurate and it may allocate too many PGs to pools that will never hold that much data.

## Checking the Warning

```bash
ceph health detail
```

Example output:

```text
[WRN] POOL_TARGET_SIZE_BYTES_OVERCOMMITTED: Total target size 12 TiB exceeds cluster raw capacity 8 TiB
```

Check what each pool's target size is set to:

```bash
for pool in $(ceph osd pool ls); do
  val=$(ceph osd pool get $pool target_size_bytes 2>/dev/null | awk '{print $2}')
  echo "$pool: $val"
done
```

Check total cluster raw capacity:

```bash
ceph df | grep TOTAL
```

## Understanding target_size_bytes

This setting is purely a hint for the autoscaler - it does not reserve capacity or limit writes. It helps the autoscaler pre-allocate an appropriate number of PGs before data is actually written to the pool.

Overcommitting does not cause data loss - but it causes the autoscaler to over-allocate PGs, leading to `TOO_MANY_PGS` warnings and wasted memory on OSDs.

## Fix Options

### Option 1 - Reduce target_size_bytes for Each Pool

Calculate realistic sizes based on expected data for each pool, ensuring the total stays within cluster capacity:

```bash
ceph osd pool set volumes target_size_bytes 2199023255552
ceph osd pool set images target_size_bytes 1099511627776
ceph osd pool set backups target_size_bytes 549755813888
```

Verify the total is within limits:

```bash
python3 -c "print((2199023255552 + 1099511627776 + 549755813888) / (1024**4), 'TiB')"
```

### Option 2 - Remove target_size_bytes and Use Ratios Instead

Switch from absolute byte targets to ratios that automatically scale with cluster size:

```bash
ceph osd pool set volumes target_size_bytes 0
ceph osd pool set volumes target_size_ratio 0.4

ceph osd pool set images target_size_bytes 0
ceph osd pool set images target_size_ratio 0.2
```

Ensure ratios across all pools do not exceed 1.0 in total:

```bash
for pool in $(ceph osd pool ls); do
  ceph osd pool get $pool target_size_ratio 2>/dev/null
done
```

### Option 3 - Remove All Target Hints

If you want the autoscaler to work purely from actual data volume:

```bash
for pool in $(ceph osd pool ls); do
  ceph osd pool set $pool target_size_bytes 0
  ceph osd pool set $pool target_size_ratio 0.0
done
```

## Summary

`POOL_TARGET_SIZE_BYTES_OVERCOMMITTED` warns that the sum of pool target size hints exceeds actual cluster capacity. Fix it by reducing or removing `target_size_bytes` values, switching to `target_size_ratio` values that sum to 1.0 or less, or removing all hints so the autoscaler works from actual data volumes. This is a non-critical warning but should be addressed to keep PG autoscaling accurate.
