# How to Monitor D3N Cache Hit Rates in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, D3N, Cache, Monitoring, RGW, Performance

Description: Monitor D3N cache hit rates in Ceph RGW using performance counters, admin socket commands, and Prometheus metrics to validate cache effectiveness.

---

## Overview

Monitoring cache hit rates is essential to verify that D3N is providing value. A low hit rate means objects are being fetched from the backend cluster repeatedly, negating the benefits of caching. This guide covers how to extract and interpret D3N hit rate metrics.

## Using the Admin Socket

The Ceph admin socket exposes real-time performance counters for RGW including D3N metrics:

```bash
# Connect to the RGW admin socket
ceph daemon rgw.myzone perf dump

# Filter for D3N specific counters
ceph daemon rgw.myzone perf dump | python3 -m json.tool | grep -A2 -i "d3n"
```

Key counters to look for:

- `d3n_cache_hit` - number of cache hits
- `d3n_cache_miss` - number of cache misses
- `d3n_cache_eviction` - number of objects evicted
- `d3n_cache_write` - number of objects written to cache

## Calculating Hit Rate

```bash
# Extract hit and miss counts
ceph daemon rgw.myzone perf dump > /tmp/d3n-perf.json

# Calculate hit rate with Python
python3 << 'EOF'
import json

with open('/tmp/d3n-perf.json') as f:
    data = json.load(f)

# Navigate the perf dump structure
for section, metrics in data.items():
    if isinstance(metrics, dict):
        hits = metrics.get('d3n_cache_hit', {}).get('val', 0)
        misses = metrics.get('d3n_cache_miss', {}).get('val', 0)
        if hits + misses > 0:
            rate = hits / (hits + misses) * 100
            print(f"Hit rate: {rate:.1f}% ({hits} hits, {misses} misses)")
EOF
```

## Monitoring with Prometheus and Grafana

Ceph exposes metrics via the MGR Prometheus module. Enable it and configure scraping:

```bash
# Enable Prometheus module
ceph mgr module enable prometheus

# Check metrics endpoint
curl http://$(ceph mgr dump | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['active_addr'].split(':')[0])" ):9283/metrics | grep d3n
```

Add a Grafana panel with this PromQL query:

```promql
rate(ceph_rgw_d3n_cache_hit[5m]) /
(rate(ceph_rgw_d3n_cache_hit[5m]) + rate(ceph_rgw_d3n_cache_miss[5m]))
```

## Interpreting Hit Rates

| Hit Rate | Assessment | Action |
|---|---|---|
| > 80% | Excellent | No action needed |
| 60-80% | Good | Monitor trends |
| 40-60% | Fair | Consider increasing cache size |
| < 40% | Poor | Review workload suitability for D3N |

## RGW Log Analysis

```bash
# Count cache operations from RGW log
journalctl -u ceph-radosgw@rgw.myzone --since "1 hour ago" --no-pager | \
    grep -c "cache hit"

journalctl -u ceph-radosgw@rgw.myzone --since "1 hour ago" --no-pager | \
    grep -c "cache miss"
```

## Resetting Counters for Clean Measurements

```bash
# Reset perf counters to start fresh measurement
ceph daemon rgw.myzone perf reset all
```

## Summary

Monitoring D3N cache hit rates requires combining admin socket perf dump data, Prometheus metrics, and RGW log analysis. Aim for a hit rate above 60% - below that threshold, consider increasing cache size, reviewing whether your workload pattern suits D3N, or checking for configuration issues. Grafana dashboards provide the best long-term visibility into cache performance trends.
