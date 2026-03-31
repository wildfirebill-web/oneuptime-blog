# How to Understand Ceph Cluster Maps (Monitor, OSD, PG, CRUSH, MDS)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, ClusterMap, Monitor, Storage, Kubernetes

Description: Learn the five Ceph cluster maps - Monitor, OSD, PG, CRUSH, and MDS - and how they coordinate distributed storage decisions across the cluster.

---

## The Role of Cluster Maps

Ceph's design relies on a set of authoritative cluster maps that describe the current state of every component. Clients, OSDs, and monitors all maintain local copies of these maps and propagate updates when state changes. No single component needs to ask a central authority for every I/O decision - instead, each actor computes its own behavior from the maps.

## The Five Core Maps

### 1. Monitor Map

The Monitor Map lists all active monitors, their IP addresses, and the current epoch (version number). Every component uses this map to locate the monitor quorum.

```bash
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph mon dump
```

### 2. OSD Map

The OSD Map tracks every OSD: whether it is up (responding) and in (included in the CRUSH map for data placement). It also records pool definitions and PG counts.

```bash
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph osd dump
```

Key fields per OSD:

```text
osd.0 up   in  weight 1.00 ...
osd.1 up   in  weight 1.00 ...
osd.2 down out weight 0.00 ...
```

An OSD that is `down out` no longer receives new writes and recovery is underway.

### 3. PG Map

The Placement Group (PG) Map tracks the state of every PG in the cluster. PGs are the unit of replication and recovery.

```bash
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph pg stat
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph pg dump_stuck
```

Common PG states include:
- `active+clean` - healthy
- `active+degraded` - some replicas missing, still serving I/O
- `peering` - OSD acting set negotiating PG state
- `recovering` - rebuilding lost replicas

### 4. CRUSH Map

The CRUSH Map defines the hierarchy of storage devices (OSDs, hosts, racks, zones) and the rules for distributing PGs across them. It is embedded within the OSD map but can be extracted independently.

```bash
kubectl exec -it rook-ceph-tools -n rook-ceph -- bash -c \
  "ceph osd getcrushmap -o /tmp/crush.bin && crushtool -d /tmp/crush.bin -o - | head -40"
```

### 5. MDS Map

The MDS Map is relevant only when CephFS is deployed. It tracks active and standby MDS daemons, which manage filesystem metadata.

```bash
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph mds stat
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph fs dump
```

## Map Epochs and Propagation

Every map has an epoch counter that increments with each change. Monitors distribute map updates to OSDs and clients as needed. You can observe epoch progression:

```bash
# Watch monitor map epochs in real time
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph -w
```

Output lines include epoch numbers:

```text
2026-03-31 10:00:01.123 mon.a [INF] osdmap e205: 12 osds: 12 up, 12 in
2026-03-31 10:00:15.456 mon.a [INF] osdmap e206: 12 osds: 11 up, 12 in
```

## Checking Map Consistency

When troubleshooting, verify that all monitors have consistent maps:

```bash
kubectl exec -it rook-ceph-tools -n rook-ceph -- ceph quorum_status | python3 -m json.tool
```

A healthy output shows all monitors in the quorum list with matching leader epoch values.

## Summary

Ceph's five cluster maps (Monitor, OSD, PG, CRUSH, and MDS) form a self-consistent distributed state system that allows every component to make autonomous storage decisions. Monitors hold the authoritative copies and distribute incremental updates. Understanding how to read and interpret these maps is fundamental to diagnosing Rook-Ceph cluster health issues, from OSD failures to PG recovery and CephFS metadata problems.
