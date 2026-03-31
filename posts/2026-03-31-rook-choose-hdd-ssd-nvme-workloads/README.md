# How to Choose Between HDD, SSD, and NVMe for Ceph Workloads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Storage, Hardware, Performance, Planning

Description: Compare HDD, SSD, and NVMe drives for Ceph deployments, covering IOPS, latency, cost per GB, and workload suitability to guide storage hardware selection.

---

## Storage Media Comparison for Ceph

Choosing the right storage media for Ceph directly impacts performance, cost, and power consumption. Each media type has distinct characteristics that match specific workload profiles.

## Performance Characteristics

| Metric | HDD (7200 RPM) | SATA SSD | NVMe SSD |
|--------|---------------|----------|----------|
| Sequential Read | 200 MB/s | 550 MB/s | 3,500 MB/s |
| Sequential Write | 150 MB/s | 520 MB/s | 3,000 MB/s |
| Random Read IOPS | 100-200 | 80,000 | 700,000 |
| Read Latency | 5-10 ms | 0.1 ms | 0.02 ms |
| Cost per GB | $0.02-0.04 | $0.08-0.15 | $0.15-0.30 |
| Power per Drive | 6-10 W | 2-4 W | 5-8 W |

## When to Choose HDD

HDDs remain the best choice for bulk cold storage and backup workloads:

- Archive data with infrequent access
- Backup target pools
- Large media files (video, images)
- Cost-optimized capacity (petabyte-scale)

Recommended Ceph configuration for HDD pools:

```bash
# Create HDD-backed pool with appropriate PG count
ceph osd pool create hdd-data 128 128
ceph osd pool set hdd-data size 3

# Tag OSDs by type using device class
ceph osd crush set-device-class hdd osd.0 osd.1 osd.2

# Create CRUSH rule targeting only HDD OSDs
ceph osd crush rule create-replicated hdd-rule default host hdd
ceph osd pool set hdd-data crush_rule hdd-rule
```

## When to Choose SATA SSD

SATA SSDs balance performance and cost for mixed workloads:

- General-purpose databases (PostgreSQL, MySQL) with moderate IOPS
- Virtual machine disks for dev/test environments
- Log aggregation and time-series data
- Caching tier in tiered storage setups

```bash
ceph osd crush set-device-class ssd osd.3 osd.4 osd.5
ceph osd crush rule create-replicated ssd-rule default host ssd
ceph osd pool create ssd-data 64 64
ceph osd pool set ssd-data crush_rule ssd-rule
```

## When to Choose NVMe

NVMe drives are required for latency-sensitive, high-IOPS workloads:

- OLTP databases (high-frequency transactions)
- Kubernetes persistent volumes for real-time applications
- Message queues and streaming data (Kafka, Pulsar)
- Any workload requiring sub-millisecond P99 latency

```bash
ceph osd crush set-device-class nvme osd.6 osd.7 osd.8
ceph osd crush rule create-replicated nvme-rule default host nvme
ceph osd pool create nvme-fast 32 32
ceph osd pool set nvme-fast crush_rule nvme-rule
```

## Hybrid Tiering Strategy

Use NVMe or SSD as BlueStore WAL/DB devices with HDD for data:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    nodes:
    - name: "node1"
      devices:
      - name: "sda"   # HDD data
        config:
          metadataDevice: nvme0n1   # NVMe for WAL/DB
      - name: "sdb"   # HDD data
        config:
          metadataDevice: nvme0n1
```

This configuration stores BlueStore metadata (WAL and RocksDB) on fast NVMe while data resides on cheaper HDDs - giving near-SSD performance at HDD cost per GB.

## Decision Framework

```bash
# Check OSD device class assignments
ceph osd tree | grep -E "osd|hdd|ssd|nvme"
ceph osd crush dump | grep device_class

# Create multiple pools for different tiers
ceph osd pool create archive 256 256  # HDD
ceph osd pool create standard 128 128 # SSD
ceph osd pool create premium 64 64    # NVMe
```

## Summary

Choose HDD for maximum capacity per dollar when IOPS and latency are not critical. Use SATA SSDs for general-purpose production workloads balancing cost and performance. Select NVMe for latency-sensitive applications requiring sub-millisecond response times. Hybrid configurations using NVMe for BlueStore metadata with HDD data drives offer an excellent middle ground that achieves near-SSD random I/O performance at much lower cost.
