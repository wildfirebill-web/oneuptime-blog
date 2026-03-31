# How to Fix Stuck PGs (Degraded, Stale, Unclean) in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Placement Group, Troubleshooting, Recovery

Description: Learn how to diagnose and resolve stuck Placement Groups in various states including degraded, stale, and unclean in Rook-managed Ceph clusters.

---

## Understanding Stuck PG States

Ceph PGs can get stuck in various transitional states. The `ceph pg dump_stuck` command identifies PGs that have not made progress within a configurable time window:

- **Stuck inactive**: PG cannot find an active primary
- **Stuck unclean**: PG has not completed recovery to full replica count
- **Stuck stale**: PG's primary OSD has not reported its status to monitors recently
- **Stuck degraded**: PG is operating below full replica count
- **Stuck undersized**: PG has fewer acting OSDs than `size` setting

## Querying Stuck PGs

```bash
# List all stuck PGs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg dump_stuck

# Filter by specific state
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg dump_stuck stale

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg dump_stuck inactive

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg dump_stuck unclean
```

## Diagnosing a Specific Stuck PG

Query detailed state for a specific PG:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg <pgid> query
```

The output shows the PG's state history, acting set (current OSDs), up set (desired OSDs), and any error messages.

## Fixing Stuck Stale PGs

Stale PGs occur when the primary OSD stops sending heartbeats. Check if the OSD is running:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg <pgid> query | grep "primary"

kubectl -n rook-ceph get pods -l app=rook-ceph-osd | grep <osd-id>
```

Restart a non-responsive OSD pod:

```bash
kubectl -n rook-ceph delete pod rook-ceph-osd-<id>-<suffix>
```

## Fixing Stuck Unclean PGs

Unclean PGs are in the process of recovery but haven't finished. Check for blocking flags:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd dump | grep -E "norecover|nobackfill|norebalance"
```

Remove any flags that might be blocking recovery:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd unset norecover

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd unset nobackfill
```

## Forcing PG Scrub to Resolve State

Sometimes forcing a scrub unsticks a PG:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg scrub <pgid>

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg deep-scrub <pgid>
```

## Monitoring Progress

Watch PG state counts over time:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch -n 5 "ceph pg stat"
```

The number of PGs in problematic states should decrease as recovery progresses.

## Summary

Stuck PGs in degraded, stale, or unclean states indicate blocked recovery processes. Use `ceph pg dump_stuck` to identify affected PGs and `ceph pg <pgid> query` for detailed diagnostics. Fix stale PGs by restarting their primary OSD. Fix unclean PGs by removing recovery flags. Force scrubs to unstick PGs in transitional states.
