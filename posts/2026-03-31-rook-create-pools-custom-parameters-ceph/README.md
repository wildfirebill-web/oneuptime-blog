# How to Create Pools with Custom Parameters in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Storage, Kubernetes, Operation

Description: Learn how to create Ceph pools with custom parameters including PG counts, replication size, CRUSH rules, compression, and application tags.

---

## Default vs Custom Pool Creation

The simplest pool creation command creates a replicated pool with default settings:

```bash
ceph osd pool create mypool replicated
```

However, production pools almost always require custom parameters: a specific number of placement groups, a targeted CRUSH rule, a replication size other than the cluster default, or compression settings. This guide covers all the key parameters you can set at creation time or immediately after.

## Specifying PG Count at Creation

Placement groups distribute data evenly across OSDs. Specify a count when creating the pool:

```bash
ceph osd pool create mypool 128 128 replicated
```

The two `128` values are the `pg_num` and `pgp_num`. They should match. Ceph 14+ includes the `pg_autoscaler` module which can manage this automatically:

```bash
ceph osd pool set mypool pg_autoscale_mode on
```

## Setting Replication Size

Set the number of replicas with `size` (total copies) and `min_size` (minimum for writes):

```bash
ceph osd pool create mypool replicated
ceph osd pool set mypool size 3
ceph osd pool set mypool min_size 2
```

A `size=3, min_size=2` configuration allows writes when 2 out of 3 OSDs are available and fails writes only when fewer than 2 are accessible, protecting against split-brain scenarios.

## Assigning a CRUSH Rule

Target specific hardware topology or device class by specifying a CRUSH rule:

```bash
# Create the pool and immediately assign a CRUSH rule
ceph osd pool create ssd-pool replicated
ceph osd pool set ssd-pool crush_rule ssd-rule
```

Or combine everything at creation using a custom CRUSH rule name:

```bash
ceph osd pool create nvme-pool 64 64 replicated nvme-rule
```

## Enabling Compression at Creation

Configure inline compression settings right after pool creation:

```bash
ceph osd pool create compressed-pool replicated
ceph osd pool set compressed-pool compression_mode aggressive
ceph osd pool set compressed-pool compression_algorithm zstd
ceph osd pool set compressed-pool compression_min_blob_size 8192
ceph osd pool set compressed-pool compression_max_blob_size 65536
```

Available `compression_mode` values: `none`, `passive`, `aggressive`, `force`. Use `aggressive` to compress all data that benefits from it.

## Tagging the Pool for an Application

Ceph tracks which application uses each pool. Always tag pools after creation:

```bash
ceph osd pool application enable mypool rbd
ceph osd pool application enable mypool rgw
ceph osd pool application enable mypool cephfs
```

You can define custom tags for application-specific pools:

```bash
ceph osd pool application enable mypool myapp
```

## Setting Pool Quotas at Creation

Limit how much data or how many objects a pool can store:

```bash
ceph osd pool set-quota mypool max_bytes $((100 * 1024 * 1024 * 1024))
ceph osd pool set-quota mypool max_objects 1000000
```

Check quota settings:

```bash
ceph osd pool get-quota mypool
```

## Complete Example: Production RGW Pool

Here is a complete sequence for creating a production RGW data pool:

```bash
# Create pool with PG count
ceph osd pool create rgw-data 256 256 replicated

# Set replication
ceph osd pool set rgw-data size 3
ceph osd pool set rgw-data min_size 2

# Assign HDD CRUSH rule
ceph osd pool set rgw-data crush_rule hdd-rule

# Enable compression
ceph osd pool set rgw-data compression_mode passive
ceph osd pool set rgw-data compression_algorithm zstd

# Tag for application
ceph osd pool application enable rgw-data rgw

# Enable autoscaler
ceph osd pool set rgw-data pg_autoscale_mode on

echo "Pool created successfully"
ceph osd pool ls detail | grep rgw-data
```

## Summary

Creating Ceph pools with custom parameters allows you to optimize storage for specific workloads. Set the replication size and min_size for fault tolerance, assign a CRUSH rule to target device classes or failure domains, enable compression for appropriate workloads, and always tag the pool with an application name. Using the pg_autoscaler reduces the need to manually tune PG counts over time.
