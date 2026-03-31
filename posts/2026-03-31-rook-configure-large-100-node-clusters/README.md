# How to Configure Ceph for Large (100+ Node) Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Large Cluster, Scale, Configuration, Performance

Description: Scale Rook-Ceph to 100+ nodes by tuning monitor count, PG autoscaling, CRUSH hierarchy, and Kubernetes API load to handle large cluster operations reliably.

---

## Scaling Challenges at 100+ Nodes

Large Ceph clusters face unique challenges: monitor memory usage grows with map history, PG count must be managed carefully, CRUSH map complexity increases, and Rook's operator can create API server pressure during reconciliation.

## Monitor Configuration for Large Clusters

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  mon:
    count: 5   # 5 mons for large clusters (more fault tolerance)
    allowMultiplePerNode: false
  mgr:
    count: 2   # Active-standby MGR for HA
```

## Limit Monitor Map History

```bash
# Prevent monitor store from growing unbounded
ceph config set mon mon_max_osdmap_epochs 500
ceph config set mon mon_min_osdmap_epochs 40

# Compact monitor store regularly
ceph tell mon.* compact
```

## CRUSH Hierarchy for Large Deployments

```bash
# Create datacenter -> rack -> host hierarchy
ceph osd crush add-bucket dc1 datacenter
ceph osd crush add-bucket rack01 rack
ceph osd crush add-bucket rack02 rack
ceph osd crush move rack01 datacenter=dc1
ceph osd crush move rack02 datacenter=dc1

# Move hosts into racks
ceph osd crush move worker-01 rack=rack01
ceph osd crush move worker-02 rack=rack01
ceph osd crush move worker-03 rack=rack02
```

## PG Autoscaler Configuration

```bash
# Enable PG autoscaler for all pools
ceph mgr module enable pg_autoscaler
ceph config set global osd_pool_default_pg_autoscale_mode on

# Set target PGs per OSD
ceph config set global mon_target_pg_per_osd 100

# Monitor autoscaler activity
ceph osd pool autoscale-status
```

## Limit Rook Operator Reconciliation Rate

```yaml
# In the Rook operator deployment, add environment variables
# to rate-limit Kubernetes API calls
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rook-ceph-operator
spec:
  template:
    spec:
      containers:
        - name: rook-ceph-operator
          env:
            - name: ROOK_OPERATOR_TIMEOUT
              value: "15m"
            - name: ROOK_LOG_LEVEL
              value: "INFO"  # Not DEBUG in production
```

## Scale Ceph Manager Modules

```bash
# Enable Prometheus metrics module
ceph mgr module enable prometheus

# Adjust MGR module thread count for large clusters
ceph config set mgr mgr_stats_period 5  # Seconds between stats updates

# Disable unused modules to reduce MGR load
ceph mgr module disable restful
ceph mgr module disable dashboard  # If using external Grafana only
```

## Large Cluster Health Monitoring

```bash
# Get cluster summary without per-OSD details
ceph status

# Monitor slow operations (common at scale)
ceph health detail | grep "slow requests"

# Check PG distribution across OSDs
ceph osd df | awk 'NR>1 {print $2}' | sort -n | uniq -c

# Verify no OSD is responsible for too many PGs
ceph osd df | sort -k2 -n | tail -10
```

## Summary

Scaling Rook-Ceph beyond 100 nodes requires 5-monitor quorum, bounded monitor map history, a multi-level CRUSH hierarchy aligned with physical rack topology, PG autoscaling, and careful management of the Rook operator's Kubernetes API footprint. These settings keep the cluster manageable and prevent cascading failures during OSD failures or rolling upgrades.
