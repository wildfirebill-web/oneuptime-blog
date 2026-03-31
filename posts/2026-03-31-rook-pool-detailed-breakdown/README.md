# How to View Detailed Pool Breakdown in Ceph (OMAP, Compression, Quotas)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Pool, OMAP, Compression

Description: Use ceph df detail to view detailed per-pool breakdowns including OMAP metadata size, compression savings, and quota consumption in Rook-Ceph clusters.

---

`ceph df detail` extends the standard `ceph df` output with additional per-pool columns that reveal OMAP metadata overhead, compression efficiency, and quota utilization. These metrics are essential for diagnosing unexpected storage consumption.

## Run ceph df detail

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph df detail
```

Sample output (abbreviated):

```text
POOL            STORED    OBJECTS  USED      %USED  MAX AVAIL  QUOTA OBJECTS  QUOTA BYTES  DIRTY  USED COMPR  UNDER COMPR
replicapool     200 GiB   51.2k    600 GiB   3.57   5 TiB      N/A            N/A          N/A    0 B         0 B
.rgw.root       10 KiB    4        30 KiB    0.00   5 TiB      N/A            N/A          N/A    0 B         0 B
myfs-metadata   1 GiB     256      3 GiB     0.02   5 TiB      N/A            N/A          N/A    0 B         0 B
```

## Understanding Additional Columns

| Column | Description |
|---|---|
| `QUOTA OBJECTS` | Maximum object count (N/A if not set) |
| `QUOTA BYTES` | Maximum bytes allowed (N/A if not set) |
| `DIRTY` | Unflushed bytes (relevant for cache tiering) |
| `USED COMPR` | Bytes saved by compression |
| `UNDER COMPR` | Total bytes subject to compression before savings |

## Analyze OMAP Data

OMAP (object map) is key-value metadata stored alongside RADOS objects. RGW uses OMAP heavily for bucket indexes. Excessive OMAP can cause OSD performance issues.

```bash
# Check OMAP usage per pool
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- bash -c "
  ceph df detail --format json | python3 -c \"
import json, sys
data = json.load(sys.stdin)
for pool in data['pools']:
    omap = pool['stats'].get('omap_bytes_used', 0)
    name = pool['name']
    if omap > 0:
        print(f'{name}: OMAP = {omap // (1024**2)} MiB')
\"
"
```

## Analyze Compression Savings

For pools with compression enabled:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- bash -c "
  ceph df detail --format json | python3 -c \"
import json, sys
data = json.load(sys.stdin)
for pool in data['pools']:
    compressed = pool['stats'].get('compress_bytes_used', 0)
    under = pool['stats'].get('compress_under_bytes', 0)
    if under > 0:
        ratio = (1 - compressed / under) * 100
        print(f'{pool[\"name\"]}: {ratio:.1f}% compression savings')
\"
"
```

## Check Pool Quotas

```bash
# View all quota settings
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool ls | while read pool; do
    echo -n "$pool: "
    ceph osd pool get "$pool" max_bytes 2>/dev/null || echo "no quota"
  done
```

Or through the detail output:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph df detail --format json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for pool in data['pools']:
    qb = pool['stats'].get('quota_bytes', 0)
    qo = pool['stats'].get('quota_objects', 0)
    if qb > 0 or qo > 0:
        used = pool['stats']['stored']
        pct = used / qb * 100 if qb > 0 else 0
        print(f'{pool[\"name\"]}: quota={qb//(1024**3)}GiB, used={used//(1024**3)}GiB ({pct:.1f}%)')
"
```

## Identify Pools with High OMAP (RGW Concern)

If you use Ceph Object Storage (RGW), large bucket indexes create significant OMAP:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  rados -p .rgw.buckets.index ls | wc -l
```

High OMAP on `.rgw.buckets.index` may indicate the need for resharding:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin bucket stats --bucket <bucket-name>
```

## Summary

`ceph df detail` provides a complete per-pool breakdown including OMAP metadata bytes, compression savings (USED COMPR vs UNDER COMPR), quota settings, and dirty (unflushed) bytes. Use it to diagnose unexpectedly high storage usage, verify that compression is providing expected savings, and track quota consumption. Parse `--format json` output for programmatic analysis across all pools.
