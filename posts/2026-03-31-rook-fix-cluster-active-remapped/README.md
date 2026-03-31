# How to Fix Cluster Stuck in active+remapped State

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Rebalancing, Troubleshooting

Description: Learn how to diagnose and resolve a Ceph cluster stuck in active+remapped state, which indicates data is misplaced and needs rebalancing.

---

## Understanding active+remapped State

`active+remapped` is a PG state that means the PG is operational (clients can read and write) but the acting set (OSDs currently serving data) differs from the up set (OSDs that should be serving data according to CRUSH). Data is accessible but not in its optimal location.

This state typically appears during:
- Cluster rebalancing after adding or removing OSDs
- OSD weight changes
- CRUSH map modifications
- OSD recovering from a crash

`active+remapped` is not a critical error, but if it persists for a long time, it indicates backfill is blocked.

## Checking remapped PG Count

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg stat | grep remapped

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph status | grep remapped
```

View the specific remapped PGs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg dump | grep remapped | head -20
```

## Why Backfill Gets Stuck

PGs stay in `active+remapped` when backfill is:

1. **Throttled**: `osd_max_backfills` is set low to protect production I/O
2. **Blocked by flags**: `nobackfill` flag is set
3. **Waiting on slow OSDs**: Target OSDs have high latency
4. **Insufficient space**: Target OSDs are too full to accept data

## Checking for Blocking Flags

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd dump | grep -E "flags|nobackfill|norebalance"
```

Remove flags if set unintentionally:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd unset nobackfill

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd unset norebalance
```

## Increasing Backfill Concurrency

Allow more parallel backfill operations to clear the remapped state faster:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_max_backfills 3

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_recovery_max_active 5
```

## Checking OSD Fullness

Verify target OSDs have available space:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd df | sort -k8 -rn | head -10
```

If OSDs are near full, add capacity before expecting backfill to complete.

## Monitoring Progress

Track the count of remapped PGs decreasing over time:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch -n 10 "ceph status | grep -E 'remapped|backfill|misplaced'"
```

## Summary

`active+remapped` PGs indicate data that needs backfilling to its optimal location. The state is non-critical as data remains accessible. Resolve by removing `nobackfill` flags, increasing `osd_max_backfills`, and ensuring target OSDs have available capacity. Monitor the `misplaced` object count decreasing toward zero as backfill completes.
