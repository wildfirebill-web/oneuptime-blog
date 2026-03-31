# How to Perform a Basic Ceph Health Check in 5 Minutes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Health Check, Operations, Monitoring

Description: Perform a complete Ceph cluster health check in 5 minutes using a focused set of commands that cover cluster state, OSD status, PG health, and disk usage.

---

A 5-minute Ceph health check is a routine procedure that any operator should be able to run before and after changes, or as part of a morning standup review. This guide gives you the minimal command set to verify cluster health quickly.

## The 5-Minute Health Check Script

Run this sequence for a complete health snapshot:

```bash
#!/bin/bash
echo "=== $(date) ==="
echo ""
echo "--- Cluster Summary ---"
ceph status
echo ""
echo "--- Health Detail (if not OK) ---"
ceph health detail
echo ""
echo "--- OSD Status ---"
ceph osd stat
echo ""
echo "--- Disk Usage ---"
ceph df
echo ""
echo "--- PG Status ---"
ceph pg stat
echo ""
echo "--- Recent Events ---"
ceph log last 10
```

## Interpreting the Output

### ceph status - The Big Picture

The most important fields:

```
  cluster:
    health: HEALTH_OK          # Should be HEALTH_OK

  services:
    mon: 3 daemons, quorum ... # All MONs should be in quorum
    osd: 12 osds: 12 up, 12 in # All OSDs should be "up" and "in"

  data:
    pools:   3 pools, 96 pgs
    pgs:     96 active+clean   # All PGs should be "active+clean"
    objects: 123 objects

  io:
    recovery: 0 B/s
```

### Disk Usage Thresholds

```bash
ceph df
```

Watch for pools approaching the `nearfull` ratio (default 85%) or `full` ratio (default 95%). When a pool goes full, writes stop.

### OSD Down/Out

```bash
ceph osd stat
# "12 osds: 11 up, 12 in" - one OSD is down, needs attention
```

Any OSD that is "down" or "out" should be investigated:

```bash
ceph osd tree | grep -E "down|out"
```

## Warning Signs to Investigate Further

```bash
# Check for stuck PGs
ceph pg dump_stuck | grep -v "^ok"

# Check for backfill/recovery activity
ceph status | grep -E "backfill|recover|degraded"

# Check for slow requests
ceph health detail | grep "slow"
```

## Running the Check via Rook

In a Kubernetes environment with Rook:

```bash
TOOLBOX=$(kubectl get pod -n rook-ceph -l app=rook-ceph-tools -o name | head -1)
kubectl exec -n rook-ceph "$TOOLBOX" -- ceph status
kubectl exec -n rook-ceph "$TOOLBOX" -- ceph health detail
kubectl exec -n rook-ceph "$TOOLBOX" -- ceph osd stat
```

## Summary

A basic 5-minute Ceph health check covers cluster status, OSD count, PG cleanliness, and disk usage. The most important single indicator is `HEALTH_OK` in `ceph status`. Anything deviating from `active+clean` PGs or having OSDs down or near-full disk usage warrants deeper investigation before proceeding with any cluster changes.
