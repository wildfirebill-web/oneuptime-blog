# How to Set DmClock Reservations for Ceph Clients

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, QoS, DmClock, Reservation

Description: Learn how to configure DmClock reservations to guarantee minimum I/O rates for specific Ceph clients, ensuring critical workloads are never starved.

---

## What are DmClock Reservations?

In the DmClock QoS model, a reservation is the minimum I/O rate (in IOPS or operations per second) that a client is guaranteed to receive, even when the OSD is under heavy load from other clients. Reservations act as a floor below which a client's I/O rate never drops.

## How Reservations Work

DmClock tracks a "reservation time" for each client. When a request arrives before the reservation time, it is assigned a reservation-based tag that gives it high scheduling priority. This ensures the client receives at least its reserved rate.

The total reservations for all clients on an OSD should not exceed 70-80% of the OSD's total IOPS capacity, leaving headroom for burst traffic.

## Setting Global Client Reservations

Set the baseline reservation for all client I/O:

```bash
# Set 1000 IOPS minimum for client operations
ceph config set osd osd_mclock_scheduler_client_res 1000
```

This applies to all OSDs unless overridden at the OSD level.

## Setting Per-Client Reservations via RBD QoS

For RBD images, set per-image I/O reservations using image metadata:

```bash
# Set minimum IOPS reservation for a specific RBD image
rbd config image set mypool/prod-db rbd_qos_iops_burst_seconds 1
rbd config image set mypool/prod-db rbd_qos_read_iops_limit 5000
rbd config image set mypool/prod-db rbd_qos_write_iops_limit 2000
```

Check image QoS settings:

```bash
rbd config image get mypool/prod-db
```

## Per-Client Reservation via Ceph Config DB

Set reservations for specific RADOS clients using the config database:

```bash
# Get the client ID from auth list
ceph auth list | grep client.

# Set reservation for a specific client
ceph config set client.db-server osd_mclock_scheduler_client_res 500
```

## Setting Reservations for CephFS Clients

For CephFS workloads, configure QoS at the MDS level:

```bash
ceph config set mds mds_max_caps_per_client 50000
ceph config set client client_caps_release_delay 5
```

## Monitoring Reservation Utilization

Check whether clients are hitting their reservation limits:

```bash
# View OSD performance counters
ceph daemon osd.0 perf dump | python3 -m json.tool | grep -A5 "mclock"
```

Monitor per-OSD IOPS to see if any client is being throttled to reservation:

```bash
ceph osd perf | sort -k3 -rn | head -10
```

## Calculating Safe Reservation Values

Use this formula to set safe reservations:

```text
Safe total reservations per OSD = OSD max IOPS x 0.75
Per-client reservation = Safe total / number of priority clients
```

For example, a 10,000 IOPS NVMe OSD serving four priority clients:

```text
Safe total = 10000 x 0.75 = 7500 IOPS
Per-client = 7500 / 4 = 1875 IOPS
```

```bash
ceph config set osd.0 osd_mclock_scheduler_client_res 1875
```

## Testing Reservation Enforcement

Verify that reserved clients maintain their minimum rate under contention:

```bash
# Create load with a background workload
rados bench -p testpool 60 write --no-cleanup -t 16 &

# Measure reserved client throughput
rados bench -p prodpool 60 write -t 4 | grep "Average IOPS"
```

The reserved client should maintain close to its guaranteed minimum rate.

## Summary

DmClock reservations give critical Ceph clients a guaranteed minimum I/O rate that is honored even under cluster-wide congestion. Setting client-level reservations through the config database, RBD image metadata, or global OSD config ensures production databases and other latency-sensitive workloads are protected from interference by background operations or less-critical clients.
