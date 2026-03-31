# How to Check PG Distribution Across OSDs in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, PG, OSD, CRUSH, Performance, Balancing

Description: Inspect Placement Group distribution across Ceph OSDs to identify imbalances, verify CRUSH behavior, and optimize data spreading.

---

## Why PG Distribution Matters

Placement Groups (PGs) are the unit Ceph uses to distribute data across OSDs. Uneven PG distribution means some OSDs hold more data, leading to capacity imbalances, hotspots, and slower recovery. Ideally, each OSD should hold approximately the same number of PGs.

## Checking PG Distribution Per OSD

The quickest view uses `ceph osd df`:

```bash
ceph osd df
```

The `PGS` column shows how many PGs each OSD is responsible for. Look for significant variation from the average.

## Detailed PG Stats

For more detail:

```bash
ceph pg dump | head -5
```

Count PGs per OSD using the pg dump output:

```bash
ceph pg dump pgs_brief | awk '{print $NF}' | tr ',' '\n' | grep -o 'osd\.[0-9]*' | sort | uniq -c | sort -rn | head -20
```

## Using the PG Balancer

Ceph Luminous and later include an automatic PG balancer. Check its status:

```bash
ceph balancer status
```

Enable it (if not already active):

```bash
ceph balancer on
ceph balancer mode upmap
```

Evaluate what the balancer would change:

```bash
ceph balancer eval
```

Apply the balancer's optimization:

```bash
ceph balancer optimize myplan
ceph balancer show myplan
ceph balancer execute myplan
```

## Manual PG Rebalancing with upmap

For surgical adjustments, use the `osd_primary_affinity` or upmap items:

```bash
# See current upmap entries
ceph osd dump | grep "pg_upmap"

# Add a specific upmap item to move a PG replica from OSD 3 to OSD 7
ceph osd pg-upmap-items 2.a4 3 7
```

## Checking PG States

Unhealthy PGs are a sign of OSD or network issues:

```bash
ceph pg stat
```

Get details on degraded or stuck PGs:

```bash
ceph pg dump_stuck
ceph health detail | grep -i "pg"
```

## Expected PG Count Per OSD

Use this formula to estimate the ideal PG count per OSD:

```text
target_PGs_per_OSD = (total_PGs * pool_size) / num_OSDs
```

For example: 256 PGs, 3 replicas, 24 OSDs = 32 PGs per OSD.

## Summary

Check PG distribution with `ceph osd df` for a quick per-OSD count and `ceph pg dump` for detailed state. Enable the `upmap` balancer with `ceph balancer on` to let Ceph automatically equalize PG distribution. Use `ceph balancer optimize` to preview changes before applying them.
