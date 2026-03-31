# How to Understand D3N (Datacenter Data Delivery Network) in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Storage, D3N, Caching, RGW, Object Storage

Description: Learn what D3N is in Ceph RGW, how it improves object storage performance through local caching, and when to use it in Rook-Ceph deployments.

---

## What is D3N

D3N (Datacenter Data Delivery Network) is a distributed read caching layer for the Ceph RADOS Gateway (RGW). It is inspired by content delivery networks but operates within a datacenter to cache frequently accessed object data on local fast storage (typically NVMe SSDs) to reduce latency and improve read throughput for object storage workloads.

D3N was introduced to address scenarios where Ceph RGW serves the same objects repeatedly - such as machine learning training data, software packages, or media assets - where caching dramatically reduces backend I/O.

## How D3N Works

```text
Client Request
      |
      v
   RGW Instance
      |
      |-- Cache Hit? --> Serve from local NVMe SSD (low latency)
      |
      |-- Cache Miss? --> Fetch from RADOS backend
                              |
                              v
                         Store in D3N Cache
                              |
                              v
                         Serve to client
```

D3N caches are local to each RGW instance. In a multi-RGW deployment, each instance maintains its own independent cache. The cache is backed by a directory on a fast local disk (NVMe recommended) and uses a configurable eviction policy (LRU or LFUDA).

## D3N vs Traditional RGW Caching

| Feature | Traditional RGW | D3N |
|---------|----------------|-----|
| Cache location | Memory (RAM) | NVMe SSD |
| Cache size limit | RAM capacity | Disk capacity (TBs) |
| Cache persistence | Lost on restart | Survives restarts |
| Read amplification | High for cold data | Low for warm data |
| Best for | Low-latency small objects | Large repeated reads |

## When to Use D3N

D3N is most beneficial when:
- The same large objects are read repeatedly (e.g., ML training datasets)
- RGW instances have local NVMe storage available
- Network bandwidth between RGW and OSD nodes is a bottleneck
- Read throughput is more important than write performance

D3N does NOT help with:
- Write-heavy workloads (only reads are cached)
- Unique object access patterns (no repeated reads)
- Workloads where latency is already within acceptable bounds

## Enabling D3N in Rook-Ceph

Configure D3N in the CephObjectStore resource:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  gateway:
    port: 80
    instances: 2
  zone:
    name: default
```

D3N configuration is applied via Ceph config:

```bash
# Enable D3N cache on RGW
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set client.rgw.my-store rgw_d3n_l1_local_datacache_enabled true

# Set cache directory (must be on fast NVMe storage)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set client.rgw.my-store rgw_d3n_l1_datacache_persistent_path /var/lib/ceph/rgw/cache

# Set cache size (e.g., 100GB)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set client.rgw.my-store rgw_d3n_l1_datacache_size 107374182400
```

## Monitoring D3N Cache Performance

Check D3N cache hit rates via the RGW admin API:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- radosgw-admin bucket stats --bucket=my-bucket
```

Or check cache metrics from RGW perf counters:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph tell rgw.* perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
for daemon, stats in data.items():
    cache_stats = {k: v for k, v in stats.items() if 'd3n' in k.lower() or 'cache' in k.lower()}
    if cache_stats:
        print(f'Daemon: {daemon}')
        for k, v in cache_stats.items():
            print(f'  {k}: {v}')
"
```

## D3N Cache Eviction Policies

D3N supports two eviction policies:

```bash
# LRU (Least Recently Used) - default, evicts oldest accessed objects
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set client.rgw.my-store rgw_d3n_l1_eviction_policy lru

# LFUDA (Least Frequently Used with Dynamic Aging) - better for skewed access
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set client.rgw.my-store rgw_d3n_l1_eviction_policy lfuda
```

LFUDA is recommended for workloads with power-law access distributions (e.g., ML training where some datasets are accessed much more than others).

## Summary

D3N is a local NVMe cache layer for Ceph RGW that dramatically improves read performance for repeated object access patterns. It is ideal for machine learning, media serving, and software distribution workloads where the same large objects are read repeatedly. Enable it by configuring `rgw_d3n_l1_local_datacache_enabled` and pointing the cache path to a fast NVMe device, then monitor cache hit rates to validate performance improvements.
