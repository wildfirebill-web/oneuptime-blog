# How to Fix TOO_FEW_PGS Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Placement Group, Health Check, Performance

Description: Learn how to fix the TOO_FEW_PGS warning in Ceph, which indicates pools have too few placement groups to efficiently distribute data across OSDs.

---

## What Is TOO_FEW_PGS?

`TOO_FEW_PGS` is a Ceph health warning that fires when a pool has fewer Placement Groups (PGs) than recommended for the current number of OSDs and the pool's replication factor. Too few PGs lead to uneven data distribution - some OSDs end up storing much more data than others - and reduced parallelism during recovery.

The recommended PG count per pool is typically calculated as:

```text
target_pgs_per_osd * num_osds / replication_factor
```

## Checking the Warning

```bash
ceph health detail
```

Example output:

```text
[WRN] TOO_FEW_PGS: 1 pool(s) do not have enough placement groups
    pool 'rbd' has 64 PGs, recommended 128
```

Check current PG counts for all pools:

```bash
ceph osd pool ls detail
ceph pg stat
```

## Calculating the Correct PG Count

Use Ceph's built-in PG calculator:

```bash
ceph osd pool get <pool-name> pg_num
ceph osd pool autoscale-status
```

For manual calculation - with 12 OSDs, replication factor 3, targeting 100 PGs per OSD:

```text
100 * 12 / 3 = 400 -> round down to nearest power of 2 = 256
```

## Fixing the PG Count

### Option 1 - Enable PG Autoscaler (Recommended)

The PG autoscaler automatically adjusts PG counts as OSDs are added or removed:

```bash
ceph mgr module enable pg_autoscaler
ceph config set global osd_pool_default_pg_autoscale_mode on
```

Set autoscale mode for an existing pool:

```bash
ceph osd pool set <pool-name> pg_autoscale_mode on
```

Monitor autoscaler recommendations:

```bash
ceph osd pool autoscale-status
```

### Option 2 - Manually Increase PG Count

Increase `pg_num` first, then `pgp_num`:

```bash
ceph osd pool set rbd pg_num 256
```

Wait for all PGs to become active+clean before increasing `pgp_num`:

```bash
watch ceph -s
ceph osd pool set rbd pgp_num 256
```

Increasing `pg_num` triggers splitting - existing PGs are divided and data is redistributed.

### Option 3 - Set a Target PG Count

```bash
ceph osd pool set <pool-name> pg_num_target 128
```

The autoscaler will gradually move toward this target.

## Monitoring PG Split Progress

```bash
ceph -s | grep pgs
ceph pg dump_stuck | wc -l
```

## Best Practice - Set Autoscale at Pool Creation

```bash
ceph osd pool create my-pool 16 16 replicated --autoscale-mode on
```

## Summary

`TOO_FEW_PGS` means a pool's PG count is too low for efficient data distribution across your OSDs. The best fix is to enable the PG autoscaler module, which dynamically manages PG counts. For manual management, calculate the optimal PG count and increase `pg_num` followed by `pgp_num`, always waiting for PG splitting to complete between changes.
