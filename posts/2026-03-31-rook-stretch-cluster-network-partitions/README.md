# How to Handle Network Partitions in Rook Stretch Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Stretch Cluster, Network Partition, High Availability

Description: Learn how to detect, handle, and recover from network partitions in Rook-Ceph stretch clusters spanning multiple data centers.

---

## Overview

Rook stretch clusters distribute Ceph monitors and OSDs across two physical sites with a tiebreaker monitor in a third site. This topology is designed for high availability but introduces unique challenges when network partitions occur between sites. Understanding how to handle these partitions is critical for maintaining data integrity and minimizing downtime.

## How Stretch Clusters Respond to Network Partitions

When a network partition isolates one site, Ceph relies on its monitor quorum to determine which site can continue serving I/O. With a 3-monitor stretch layout (one per site plus a tiebreaker), the remaining two monitors form a quorum and allow the surviving site to continue operating.

The tiebreaker monitor is crucial - it breaks ties during a split. Without it, both sites would lose quorum and I/O would halt.

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  mon:
    count: 5
    stretchCluster:
      failureDomainLabel: topology.kubernetes.io/zone
      subFailureDomain: host
      zones:
      - name: zone-a
        arbiter: false
      - name: zone-b
        arbiter: false
      - name: zone-arbiter
        arbiter: true
```

## Detecting a Network Partition

Check the Ceph monitor status and quorum health immediately when connectivity issues arise:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon stat
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph quorum_status -f json-pretty
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

A partition will typically show `HEALTH_WARN` or `HEALTH_ERR` with messages about down monitors or OSDs.

## Handling OSD Connectivity Loss During Partition

When a site loses connectivity, its OSDs are marked down. The cluster enters a degraded state. Depending on pool replication settings, I/O may continue on the surviving side:

```bash
# Check OSD status across zones
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree

# Check which OSDs are down
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd df tree

# Review degraded placement groups
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph pg stat
```

## Configuring OSD Down-out Interval

You can tune how quickly Ceph marks isolated OSDs as `out` after a partition:

```bash
# Set the interval before OSDs are marked out (seconds)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_heartbeat_grace 20

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mon mon_osd_down_out_interval 600
```

Increasing `mon_osd_down_out_interval` prevents premature rebalancing during short outages, which reduces unnecessary network traffic and potential data movement.

## Recovery After Partition Heals

Once connectivity is restored, monitors re-join quorum automatically. OSDs that were previously isolated begin peering and the cluster starts recovering degraded placement groups:

```bash
# Monitor recovery progress
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph -w

# Check recovery I/O rates
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd pool stats
```

## Preventing Data Loss with stretch_rule Pools

Pools used in stretch clusters should use the stretch CRUSH rule, which ensures data is replicated to both sites:

```bash
# Verify pool CRUSH rule
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool get rbd crush_rule
```

Pools without the stretch rule may only write to one site, making them vulnerable to full data loss if that site goes down.

## Summary

Handling network partitions in Rook stretch clusters requires understanding monitor quorum mechanics, OSD heartbeat configuration, and pool-level CRUSH rules. The tiebreaker arbiter monitor is essential for breaking site-level ties. Tuning `mon_osd_down_out_interval` prevents excessive rebalancing during transient partitions, and verifying stretch CRUSH rules on all pools ensures both sites hold copies of your data.
