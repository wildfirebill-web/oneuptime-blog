# How to List Pools with ceph osd pool ls and ceph osd lspools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Pool Management, CLI, Storage Administration

Description: Use ceph osd pool ls, ceph osd lspools, and related commands to list, inspect, and audit Ceph storage pools with their configurations and statistics.

---

## Basic Pool Listing Commands

Ceph provides several commands for listing pools, each with different output formats and detail levels.

### ceph osd pool ls

The simplest way to list all pools:

```bash
# List pool names only
ceph osd pool ls

# List with additional detail (type, size, etc.)
ceph osd pool ls detail
```

Example output of `ceph osd pool ls`:

```text
device_health_metrics
.mgr
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
default.rgw.buckets.index
default.rgw.buckets.data
cephfs-metadata
cephfs-data
mypool
```

### ceph osd lspools

An older but still functional alias:

```bash
ceph osd lspools
```

Output includes the pool ID and name:

```text
1 device_health_metrics
2 .mgr
3 .rgw.root
4 default.rgw.log
5 default.rgw.control
6 cephfs-metadata
7 cephfs-data
8 mypool
```

## Detailed Pool Information

### osd pool ls detail

Shows pool type, size, CRUSH rule, and other configuration:

```bash
ceph osd pool ls detail
```

Example output:

```text
pool 1 'device_health_metrics' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 1 pgp_num 1 autoscale_mode on last_change 12 flags hashpspool stripe_width 0 application mgr_devicehealth
pool 8 'mypool' replicated size 3 min_size 2 crush_rule 0 object_hash rjenkins pg_num 128 pgp_num 128 autoscale_mode on
```

## Getting Pool Statistics

### ceph df

Shows used and available space per pool:

```bash
ceph df

# Detailed view with raw bytes
ceph df detail
```

Example output:

```text
--- POOLS ---
POOL                     ID  PGS   STORED  OBJECTS  USED    %USED  MAX AVAIL
device_health_metrics     1    1    0 B       0      0 B       0     450 GiB
.mgr                      2    1  960 KiB     4    2.8 MiB     0     450 GiB
cephfs-metadata           6   64   10 MiB   228     30 MiB     0     450 GiB
cephfs-data               7  256   25 GiB  6400     75 GiB    5     450 GiB
mypool                    8  128    1 GiB   256      3 GiB     0     450 GiB
```

### ceph osd pool stats

Per-pool I/O statistics:

```bash
# All pools
ceph osd pool stats

# Specific pool
ceph osd pool stats mypool
```

## Getting Specific Pool Settings

Use `ceph osd pool get` to query individual settings:

```bash
# Get pool size (replication factor)
ceph osd pool get mypool size

# Get min_size
ceph osd pool get mypool min_size

# Get PG count
ceph osd pool get mypool pg_num

# Get CRUSH rule
ceph osd pool get mypool crush_rule

# Get all pool settings at once
ceph osd pool get mypool all
```

## Listing Pools with Application Tags

Pools can be tagged with the application using them:

```bash
# View application tags per pool
ceph osd pool application get

# Or for a specific pool
ceph osd pool application get mypool
```

Output:

```text
{
    "cephfs": {},
    "rbd": {},
    "rgw": {}
}
```

## JSON Output for Scripting

Use `--format json` or `--format json-pretty` for machine-parseable output:

```bash
# List pools as JSON
ceph osd pool ls detail --format json-pretty

# Get pool stats as JSON
ceph df --format json-pretty | jq '.pools[] | {name: .name, stored: .stats.stored, used: .stats.bytes_used}'

# List all pool names in a script
pools=$(ceph osd pool ls --format json | python3 -c "import sys,json; print('\n'.join(json.load(sys.stdin)))")
echo "$pools"
```

## Listing Pools via OSD Dump

The `ceph osd dump` command includes full pool configuration:

```bash
# Full OSD and pool dump
ceph osd dump

# Extract just pool information
ceph osd dump --format json-pretty | jq '.pools[] | {pool_name, type, size, min_size, pg_num}'
```

## Audit Script Example

Script to audit all pools and their key settings:

```bash
#!/bin/bash
echo "Pool Name | ID | Type | Size | Min Size | PGs | CRUSH Rule | Application"
echo "---------|----|----|------|----------|-----|------------|------------"

ceph osd pool ls detail --format json 2>/dev/null | python3 -c "
import sys, json
pools = json.load(sys.stdin)
for p in pools:
    print(f\"{p['pool_name']} | {p['pool_id']} | {p['type']} | {p['size']} | {p['min_size']} | {p['pg_num']} | {p['crush_rule']} | {list(p.get('application_metadata', {}).keys())}\")
"
```

## Summary

Ceph provides `ceph osd pool ls` and `ceph osd lspools` as the primary commands for listing pools. Adding the `detail` flag to `pool ls` shows configuration like size, min_size, and PG count. `ceph df` and `ceph osd pool stats` provide capacity and I/O statistics per pool. For scripting and automation, the `--format json` flag combined with `jq` enables flexible pool auditing and monitoring.
