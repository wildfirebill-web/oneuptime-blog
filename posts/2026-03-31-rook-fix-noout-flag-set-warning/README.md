# How to Fix "noout flag is set" Warning in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, noout, Flag, OSD, Maintenance

Description: Understand and clear the Ceph "noout flag is set" warning after maintenance operations to restore automatic OSD recovery behavior.

---

## Introduction

The `noout` flag prevents Ceph from marking OSDs as `out` when they go down, which stops automatic data recovery/rebalancing. It is commonly set during maintenance but forgotten. A forgotten `noout` flag leaves the cluster in a vulnerable state where OSD failures do not trigger recovery.

## Identifying the Warning

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Example:

```
HEALTH_WARN noout flag(s) set
```

Check all set flags:

```bash
ceph osd dump | grep flags
```

Example output:

```
flags noout,noin
```

## When noout Is Intentionally Set

The `noout` flag is useful during planned maintenance to prevent unnecessary data rebalancing:

```bash
# Set noout before maintenance
ceph osd set noout

# Perform maintenance (node reboots, disk replacements, etc.)
kubectl drain worker-2 --delete-emptydir-data --ignore-daemonsets

# After maintenance, unset noout
ceph osd unset noout
```

## Clearing the Flag

If `noout` was left set inadvertently after maintenance:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd unset noout
```

Verify it was cleared:

```bash
ceph osd dump | grep flags
ceph health
```

The warning should disappear and the cluster should return to normal recovery behavior.

## Checking for Other Stuck Flags

```bash
ceph osd dump | grep "^flags"
```

Common flags that should not be left set in production:

- `noout` - Prevents OSDs from being marked out
- `noin` - Prevents OSDs from being marked in
- `nodown` - Prevents OSDs from being marked down
- `norebalance` - Stops data rebalancing
- `norecover` - Stops data recovery
- `nobackfill` - Stops backfill operations

Clear all maintenance flags after work is complete:

```bash
for flag in noout noin nodown norebalance norecover nobackfill; do
  ceph osd unset $flag
done
```

## Automating Flag Management

Use a script for safe maintenance windows:

```bash
#!/bin/bash
# maintenance-start.sh
echo "Setting maintenance flags..."
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd set noout

echo "Maintenance window started. Remember to run maintenance-end.sh"
```

```bash
#!/bin/bash
# maintenance-end.sh
echo "Clearing maintenance flags..."
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd unset noout

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health
```

## Configuring Automatic Alerts

Monitor for flag warnings via Prometheus:

```
ceph_health_status == 2  # HEALTH_ERR
ceph_health_detail{type="OSDMAP_FLAGS"} > 0
```

## Summary

The "noout flag is set" warning means Ceph will not automatically recover from OSD failures, leaving data durability at risk. After planned maintenance, always unset the `noout` flag with `ceph osd unset noout`. Automating maintenance scripts with paired set/unset operations prevents this flag from being left on accidentally in production clusters.
