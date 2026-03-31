# How to Plan Cost-Effective Ceph Cluster Expansion

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Capacity Planning, Cost, Expansion, Storage

Description: Plan Ceph cluster expansion cost-effectively by forecasting capacity needs, timing purchases, and choosing the right hardware additions to avoid waste.

---

## Why Expansion Planning Matters

Over-provisioning Ceph clusters wastes capital. Under-provisioning causes operational emergencies. A disciplined expansion model uses utilization trends to predict when and what to buy.

## Monitor Capacity Trends

Export daily utilization data to build a trend model:

```bash
# Get current utilization snapshot
ceph df | grep TOTAL

# Script to log daily utilization (add to cron)
#!/bin/bash
DATE=$(date +%Y-%m-%d)
USED=$(ceph df --format json | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(d['stats']['total_used_bytes'])
")
echo "$DATE,$USED" >> /var/log/ceph-capacity.csv
```

## Calculate Growth Rate

```bash
# Analyze 90-day trend from CSV
python3 << 'EOF'
import csv
from datetime import datetime

rows = []
with open('/var/log/ceph-capacity.csv') as f:
    for line in csv.reader(f):
        rows.append((datetime.fromisoformat(line[0]), int(line[1])))

# Linear growth rate (bytes per day)
days = (rows[-1][0] - rows[0][0]).days
growth_per_day = (rows[-1][1] - rows[0][1]) / days
growth_tb_month = growth_per_day * 30 / (1024**4)
print(f"Growth rate: {growth_tb_month:.1f} TB/month")
EOF
```

## Determine Expansion Trigger Points

Set thresholds that give lead time for procurement:

```bash
# Alert at 70% full - this gives time to order and receive hardware
ceph osd set-full-ratio 0.95
ceph osd set-backfillfull-ratio 0.90
ceph osd set-nearfull-ratio 0.85

# Add custom alert in Prometheus
# ceph_cluster_total_used_bytes / ceph_cluster_total_bytes > 0.70
```

## Size the Expansion Correctly

```bash
# Current usable: 160 TB
# Growth rate: 5 TB/month
# Time to 70% full at current rate: (160 * 0.70 - current_used) / 5
# If currently 70 TB used: (112 - 70) / 5 = 8.4 months

# Expansion target: buy enough for 18 months of headroom
# Additional TB needed: 5 TB/month x 18 months = 90 TB usable
# With 3x replication: 90 x 3 = 270 TB raw to purchase
# At 4 TB drives: 68 drives needed
# 4 drives per new node: ~17 new nodes
```

## Choose Add-Nodes vs. Add-Drives

Adding drives to existing nodes is cheaper but increases failure domain risk:

```yaml
# Adding drives to existing node - Rook detects automatically
# Edit node device list in CephCluster spec
spec:
  storage:
    nodes:
      - name: "worker-01"
        devices:
          - name: "sdb"
          - name: "sdc"
          - name: "sdd"  # New drive added
          - name: "sde"  # New drive added
```

## Rebalancing After Expansion

```bash
# After adding OSDs, monitor rebalancing progress
watch -n5 ceph status

# Throttle rebalancing to avoid impacting workloads
ceph osd set-backfillfull-ratio 0.85
ceph config set osd osd_max_backfills 1
ceph config set osd osd_recovery_max_active 1
```

## Summary

Cost-effective Ceph expansion requires tracking utilization trends, setting early-warning thresholds, and sizing purchases to cover 12-18 months of growth. Ordering at 70% full ensures procurement lead time without the waste of buying capacity too early.
