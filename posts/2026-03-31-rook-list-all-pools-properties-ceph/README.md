# How to List All Pools and Their Properties in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Pool, Storage, Configuration, CRUSH

Description: List all Ceph storage pools and inspect their replication factor, PG count, erasure coding profile, and other properties using the CLI.

---

## Listing All Pools

Get a simple list of all pool names:

```bash
ceph osd lspools
```

For a compact view with IDs:

```bash
ceph df | grep -A 100 "POOLS"
```

## Detailed Pool Properties

To see all configuration options for every pool:

```bash
ceph osd dump | grep "^pool"
```

For a more readable format on a specific pool:

```bash
ceph osd pool get <pool-name> all
```

Example:

```bash
ceph osd pool get replicapool all
```

Sample output:

```yaml
size: 3
min_size: 2
pg_num: 128
pgp_num: 128
crush_rule: replicated_rule
hashpspool: true
nodelete: false
nopgchange: false
nosizechange: false
write_fadvise_dontneed: false
noscrub: false
nodeep-scrub: false
use_gmt_hitset: 1
fast_read: 0
```

## Key Properties Explained

| Property | Description |
|----------|-------------|
| `size` | Number of replicas |
| `min_size` | Minimum replicas required for I/O |
| `pg_num` | Number of placement groups |
| `crush_rule` | CRUSH rule controlling data placement |
| `nodelete` | Prevents pool deletion when set to true |

## Listing Erasure Code Profiles

```bash
ceph osd erasure-code-profile ls
ceph osd erasure-code-profile get default
```

## Checking Pool Quota

```bash
ceph osd pool get-quota <pool-name>
```

## Pool Stats (Usage)

```bash
ceph osd pool stats
```

Or for a specific pool:

```bash
ceph osd pool stats <pool-name>
```

## Listing Pool Application Tags

Pools are tagged with their application type (rbd, cephfs, rgw):

```bash
ceph osd pool application get <pool-name>
```

List all pools with their application tags:

```bash
ceph osd lspools | while read id name; do
  app=$(ceph osd pool application get "$name" 2>/dev/null | python3 -c "import json,sys; d=json.load(sys.stdin); print(','.join(d.keys()))" 2>/dev/null)
  echo "$name: $app"
done
```

## Checking Autoscaling

With PG autoscaling enabled, check each pool's autoscale mode:

```bash
ceph osd pool autoscale-status
```

## Summary

Use `ceph osd lspools` for a quick pool list, `ceph osd pool get <pool> all` for a full property dump, and `ceph osd pool stats` for usage metrics. Check erasure code profiles with `ceph osd erasure-code-profile ls` and monitor PG autoscaling status with `ceph osd pool autoscale-status` to ensure pools are sized correctly.
