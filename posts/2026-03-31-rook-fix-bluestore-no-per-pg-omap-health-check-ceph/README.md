# How to Fix BLUESTORE_NO_PER_PG_OMAP Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, BlueStore, OMAP, Placement Group

Description: Learn how to resolve the BLUESTORE_NO_PER_PG_OMAP health warning in Ceph by migrating OSD OMAP storage to per-PG namespaces for improved performance and isolation.

---

## Understanding BLUESTORE_NO_PER_PG_OMAP

`BLUESTORE_NO_PER_PG_OMAP` is a follow-on improvement to `BLUESTORE_NO_PER_POOL_OMAP`. While per-pool OMAP separates OMAP by pool, per-PG OMAP goes further by segregating OMAP data by individual Placement Group (PG). This was introduced in Ceph Pacific (16.x) and provides finer-grained OMAP management, enabling more efficient recovery and garbage collection at the PG level.

Check current health:

```bash
ceph health detail
```

Example output:

```text
HEALTH_WARN OSDs are not using per-pg omap
[WRN] BLUESTORE_NO_PER_PG_OMAP: 12 OSDs are not using per-pg omap
    osd.0 - osd.11 have not been converted to per-pg omap
```

## Prerequisites

Per-PG OMAP requires that per-pool OMAP is already migrated. Check that the per-pool migration is complete first:

```bash
ceph health detail | grep BLUESTORE_NO_PER_POOL_OMAP
```

If the per-pool warning still appears, complete that migration before working on per-PG OMAP.

## Enabling Per-PG OMAP

Enable the per-PG OMAP feature flag on the cluster:

```bash
ceph osd set-require-min-compat-client pacific
```

Then enable the per-PG OMAP config:

```bash
ceph config set osd bluestore_use_per_pg_omap true
```

## Triggering Migration via Deep Scrub

Like per-pool OMAP, per-PG OMAP migration happens during deep scrubs:

```bash
# Trigger deep scrub on all OSDs
ceph osd deep-scrub all
```

Monitor scrub progress:

```bash
watch "ceph pg stat"
ceph -w | grep scrub
```

## Checking Migration Status

Check which OSDs have completed per-PG OMAP migration:

```bash
ceph osd metadata | python3 -m json.tool | grep -i "per_pg_omap\|id"
```

Or check via health detail as migration progresses:

```bash
ceph health detail | grep BLUESTORE_NO_PER_PG_OMAP
```

The count of unconverted OSDs decreases as deep scrubs complete.

## Speeding Up Migration

On production clusters, accelerate scrubbing during maintenance windows:

```bash
# Increase concurrent scrubs temporarily
ceph config set osd osd_max_scrubs 4

# Allow scrubbing at any time (disable time restrictions)
ceph config set osd osd_scrub_begin_hour 0
ceph config set osd osd_scrub_end_hour 24

# Force immediate scrubs
for pg in $(ceph pg ls | grep active | awk '{print $1}'); do
  ceph pg deep-scrub $pg
done
```

Restore settings after migration:

```bash
ceph config rm osd osd_max_scrubs
ceph config rm osd osd_scrub_begin_hour
ceph config rm osd osd_scrub_end_hour
```

## Rook Deployments

Configure the per-PG OMAP setting via CephCluster config override:

```yaml
spec:
  cephConfig:
    osd:
      bluestore_use_per_pg_omap: "true"
```

Apply and trigger scrubs via toolbox:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- ceph osd deep-scrub all
```

## Benefits of Per-PG OMAP

After migration:
- PG splitting and merging is more efficient
- OSD recovery only processes relevant OMAP data
- CephFS directory operations perform better at scale
- RGW bucket index operations are isolated by PG

## Verifying Completion

```bash
ceph health detail
ceph osd metadata | python3 -m json.tool | grep per_pg_omap
```

Once all OSDs complete migration, the health warning disappears.

## Summary

`BLUESTORE_NO_PER_PG_OMAP` warns that OSDs are not yet using the per-PG OMAP storage format. First complete any per-pool OMAP migration, then enable `bluestore_use_per_pg_omap` and trigger deep scrubs to migrate. Like per-pool migration, this happens progressively as PGs are scrubbed. Temporarily increase scrub parallelism during maintenance windows to accelerate the process on large clusters.
