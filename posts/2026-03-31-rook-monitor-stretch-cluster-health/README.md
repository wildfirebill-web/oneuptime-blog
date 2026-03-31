# How to Monitor Stretch Cluster Health in Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Stretch Cluster, Monitoring, Health Check

Description: Learn how to monitor the health of Rook-Ceph stretch clusters, including quorum status, OSD health per zone, and placement group recovery metrics.

---

## Overview

A Rook-Ceph stretch cluster spans two data center sites plus a tiebreaker zone. Monitoring health in this topology goes beyond a standard single-site cluster - you need to track per-zone OSD status, monitor quorum membership, placement group distribution, and replication lag across zones. This guide walks through the key health checks and monitoring commands for stretch deployments.

## Checking Overall Cluster Health

Start with the global health summary:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

The output shows active monitors, OSD count, placement group states, and any active warnings. For a healthy stretch cluster you expect to see all 5 monitors active and all OSDs `up` and `in`.

## Verifying Monitor Quorum Per Zone

The stretch cluster relies on monitors distributed across three zones. Confirm quorum membership and which zone each monitor belongs to:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon stat
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph quorum_status -f json-pretty | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(d['quorum_names'])"
```

## Checking OSD Status by Availability Zone

Use the OSD tree to view which OSDs belong to which zone and whether they are up:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
```

Look for OSDs showing `down` status - these indicate nodes or zones that are unreachable. You can also filter by failure domain:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd crush tree --show-shadow
```

## Monitoring Placement Group Distribution

Stretch clusters use a custom CRUSH rule to guarantee data replication across both sites. Verify that placement groups are distributed correctly:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph pg dump summary
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph pg stat
```

If placement groups show `stale`, `peering`, or `incomplete` states, that is a signal that cross-site connectivity may be degraded.

## Prometheus Metrics for Stretch Clusters

Rook exposes Ceph metrics via Prometheus. Key metrics to watch for stretch deployments:

```text
ceph_mon_quorum_status          - 1 if in quorum, 0 if not
ceph_osd_up                     - OSD up/down state
ceph_pg_total                   - Total PG count
ceph_pg_active                  - Active PG count
ceph_pg_degraded                - Degraded PG count
ceph_cluster_total_bytes        - Cluster capacity
ceph_cluster_total_used_bytes   - Used capacity
```

You can query these with PromQL to build per-zone dashboards:

```bash
# Query degraded PGs
kubectl -n monitoring exec -it deploy/prometheus -- \
  curl -s 'http://localhost:9090/api/v1/query?query=ceph_pg_degraded'
```

## Setting Up Alerts

Add Prometheus alerting rules for stretch cluster health events:

```yaml
groups:
- name: rook-stretch
  rules:
  - alert: CephStretchMonQuorumLost
    expr: ceph_mon_quorum_status == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Ceph monitor lost quorum in stretch cluster"
  - alert: CephStretchOSDsDown
    expr: ceph_osd_up == 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "One or more Ceph OSDs are down"
```

## Summary

Monitoring a Rook-Ceph stretch cluster requires checking monitor quorum per zone, OSD availability across sites, and placement group distribution. Prometheus metrics combined with targeted alerting rules give you early warning of zone-level degradation. Regular use of `ceph osd tree` and `ceph status` helps verify that both sites contribute equally to the cluster's replication requirements.
