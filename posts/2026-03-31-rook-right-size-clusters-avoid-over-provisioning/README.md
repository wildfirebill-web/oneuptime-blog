# How to Right-Size Ceph Clusters to Avoid Over-Provisioning

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Capacity Planning, Cost, Right-Sizing, Storage

Description: Right-size Ceph clusters by analyzing actual utilization, forecasting growth accurately, and buying storage incrementally to avoid costly over-provisioning.

---

## The Over-Provisioning Problem

Many teams provision Ceph clusters for "future needs" and end up with 60-80% idle capacity that costs money but delivers no value. Right-sizing means buying just enough capacity to cover current needs plus a 12-18 month runway.

## Analyze Current Utilization

```bash
# Get cluster-wide utilization
ceph df

# Per-pool breakdown
ceph df detail

# OSD-level utilization (check for imbalance)
ceph osd df tree

# Check if cluster is well-balanced (all OSDs at similar %)
ceph osd df | awk 'NR>1 {print $7}' | sort -n | uniq -c
```

## Identify Idle Capacity

```bash
# Find pools with quotas set too high
ceph df detail --format json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for p in data['pools']:
    used = p['stats']['bytes_used']
    max_b = p['stats'].get('max_bytes', 0)
    name = p['name']
    if max_b > 0:
        pct = used / max_b * 100
        print(f'{name}: {pct:.1f}% used ({used/(1024**3):.0f}GB / {max_b/(1024**3):.0f}GB)')
"
```

## Forecast Growth with Historical Data

Use Prometheus and Grafana to project capacity needs:

```bash
# PromQL query to project 90-day growth
# predict_linear(ceph_cluster_total_used_bytes[30d], 90 * 24 * 3600)

# Or simple bash trend calculation
USED_NOW=$(ceph df --format json | python3 -c "
import json, sys; d=json.load(sys.stdin); print(d['stats']['total_used_bytes'])
")
# Compare with value 30 days ago from your monitoring system
# Extrapolate 12-month need from 30-day growth
```

## Set Realistic Capacity Targets

```bash
# Ceph best practices:
# Never exceed 70-80% utilization (leave room for rebalancing)
# Target 50-60% for comfortable operations

ceph osd set-nearfull-ratio 0.75
ceph osd set-backfillfull-ratio 0.85
ceph osd set-full-ratio 0.95

# If currently at 40%, you have substantial headroom
# Delay hardware purchases until you reach 60-65%
```

## Incremental Expansion Strategy

```yaml
# Instead of buying 200 TB up front, buy in increments:
# Month 1: Deploy with 60 TB usable (enough for 12 months)
# Month 13: Add 30 TB to cover next 12 months
# Month 25: Add 30 TB again

# Adding OSDs in Rook - add nodes to CephCluster spec
spec:
  storage:
    nodes:
      - name: "worker-04"   # New node added
        devices:
          - name: "sdb"
          - name: "sdc"
```

## Check OSD Memory Usage

Right-sizing RAM avoids waste without hurting performance:

```bash
# Check actual BlueStore cache memory usage
ceph daemon osd.0 perf dump | python3 -c "
import json, sys
d = json.load(sys.stdin)
cache_bytes = d['bluestore']['bluestore_cache_bytes']
print(f'OSD 0 cache: {cache_bytes/(1024**2):.0f} MB')
"

# Default is 1 GB per OSD - reduce if utilization is low
ceph config set osd bluestore_cache_size_hdd 536870912  # 512 MB for HDDs
```

## Remove Unused Pools

Unused pools waste CRUSH map complexity and monitoring overhead:

```bash
# List pools with zero data
ceph df | awk '$4 == "0" && $5 == "0" {print $1}'

# Remove confirmed empty pools
ceph osd pool delete unused-pool unused-pool --yes-i-really-really-mean-it
```

## Summary

Right-sizing Ceph clusters requires measuring actual utilization, projecting growth from real data (not guesses), and adopting incremental expansion to match purchases to need. Setting nearfull ratios at 75% and buying storage in smaller batches reduces idle hardware costs while maintaining safe operational headroom.
