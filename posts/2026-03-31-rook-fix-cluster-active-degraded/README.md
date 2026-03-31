# How to Fix Cluster Stuck in active+degraded State

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Recovery, Troubleshooting

Description: Learn how to diagnose and resolve a Ceph cluster stuck in active+degraded state, restoring full data replication and cluster health.

---

## Understanding active+degraded State

`active+degraded` means a PG is serving client requests (active) but has fewer replicas than the pool's `size` setting (degraded). Data is accessible but not fully protected. If an additional OSD fails while PGs are degraded, some data may become unavailable or even lost if the pool's `min_size` is breached.

This is more urgent than `active+remapped` because redundancy is actually reduced. The cluster will emit `HEALTH_WARN` or `HEALTH_ERR` depending on severity.

## Assessing Degraded Severity

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph status

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail | grep -E "PG_DEGRADED|degraded"
```

Check how many objects are degraded:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg stat | grep degraded
```

## Identifying Which OSDs Are Missing

Find which OSDs are down, causing the degraded state:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd tree | grep down

kubectl -n rook-ceph get pods -l app=rook-ceph-osd | grep -v Running
```

## Fixing a Down OSD Pod

If an OSD pod is crashing or in error state, restart it:

```bash
kubectl -n rook-ceph describe pod rook-ceph-osd-<id>-<suffix>
kubectl -n rook-ceph logs rook-ceph-osd-<id>-<suffix> --previous | tail -50
kubectl -n rook-ceph delete pod rook-ceph-osd-<id>-<suffix>
```

Monitor recovery after the OSD restarts:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch -n 5 ceph status
```

## Fixing a Permanently Failed OSD

If the OSD's disk has failed and cannot be recovered, mark the OSD out to trigger recovery to remaining OSDs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd out osd.<id>
```

Ceph will begin recovery, moving data from the failed OSD's PGs to surviving OSDs. Monitor recovery:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch ceph status
```

## Accelerating Recovery

Speed up recovery to reduce the window of vulnerability:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_recovery_max_active_hdd 5

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_max_backfills 3
```

## Checking min_size Safety

Ensure no PGs are below `min_size` (which causes them to become inactive):

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd dump | grep -E "min_size|size" | grep -v "^$"
```

## Summary

`active+degraded` PGs have reduced redundancy and require prompt attention. Identify down OSDs via `ceph osd tree` and restart their pods. For permanently failed OSDs, mark them out to trigger recovery to other OSDs. Accelerate recovery with higher `osd_recovery_max_active` settings to minimize the time your cluster operates with reduced redundancy.
