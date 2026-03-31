# How to Fix POOL_TOO_MANY_PGS Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Pool, Placement Group, Health Check

Description: Learn how to resolve POOL_TOO_MANY_PGS in Ceph, a per-pool warning indicating a specific pool has more PGs than its data volume warrants.

---

## What Is POOL_TOO_MANY_PGS?

`POOL_TOO_MANY_PGS` is a Ceph health warning that indicates a specific pool has been allocated more PGs than are necessary or efficient for the amount of data it currently contains. While `TOO_MANY_PGS` is about the total PG count across all pools, `POOL_TOO_MANY_PGS` identifies individual over-provisioned pools.

Over-provisioned PGs cause unnecessary memory consumption on every OSD that hosts PGs from that pool, and can result in very small PGs that do not amortize the per-PG overhead efficiently.

## Checking the Warning

```bash
ceph health detail
```

Example output:

```text
[WRN] POOL_TOO_MANY_PGS: Pool 'test-pool' has too many PGs (256)
    Recommended PG count: 32
```

Review the autoscaler's analysis:

```bash
ceph osd pool autoscale-status
```

Check pool utilization vs PG count:

```bash
ceph df detail
```

## Identifying the Root Cause

Pools become over-PG'd when:
- They were created with a high PG count but never filled with data
- Data was deleted but PG count was never reduced
- The autoscaler is not enabled

List pools with their utilization:

```bash
ceph osd pool ls detail
ceph pg dump pools | awk '{print $1, $2, $16}'
```

## Fix Steps

### Option 1 - Enable the PG Autoscaler

```bash
ceph mgr module enable pg_autoscaler
ceph osd pool set test-pool pg_autoscale_mode on
```

The autoscaler will gradually reduce PG counts for this pool.

### Option 2 - Manually Reduce pg_num

Set `pg_num` to the autoscaler's recommended value:

```bash
ceph osd pool get test-pool pg_num
ceph osd pool set test-pool pg_num 32
```

Wait for PG merging to complete - this takes longer than splitting:

```bash
watch ceph -s
```

Then reduce `pgp_num`:

```bash
ceph osd pool set test-pool pgp_num 32
```

### Option 3 - Set a Target PG Number

If you want gradual reduction:

```bash
ceph osd pool set test-pool pg_num_target 32
```

The cluster will gradually merge PGs toward the target.

## Monitoring PG Merging

PG merging (reducing `pg_num`) was added in Ceph Nautilus and involves complex state transitions. Monitor carefully:

```bash
ceph pg stat
ceph health detail | grep merging
```

Do not reduce `pg_num` while the cluster is already recovering or backfilling:

```bash
ceph -s | grep -E "degraded|recovering|backfilling"
```

## Best Practice - Set PG Count at Pool Creation

Avoid this problem by letting the autoscaler manage PG counts from pool creation:

```bash
ceph osd pool create my-pool 1 1 replicated --autoscale-mode on
```

The autoscaler starts with 1 PG and grows as data is added.

## Summary

`POOL_TOO_MANY_PGS` identifies pools with excessive PG counts relative to their actual data volume. The cleanest fix is enabling the PG autoscaler, which right-sizes PG counts automatically based on data volume. For manual management, check the autoscaler's recommended PG count and reduce `pg_num` accordingly, waiting for PG merging to complete before updating `pgp_num`.
