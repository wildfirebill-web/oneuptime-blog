# How to Write an Example ceph.conf for Production

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Configuration, Production, Performance

Description: A practical example ceph.conf file for production Ceph clusters, covering network, OSD, monitor, and performance settings.

---

## The Role of ceph.conf

The `ceph.conf` file is the central configuration file for a Ceph cluster. It defines how monitors, OSDs, and MDS daemons behave. In modern Ceph (Nautilus and later), most settings can be stored in the monitor database via `ceph config set`, but a minimal `ceph.conf` is still needed to bootstrap the cluster.

In Rook, you inject configuration through the `CephCluster` CR's `cephConfig` field rather than editing a file directly. Understanding what goes into a production `ceph.conf` helps you translate these settings into Rook configuration.

## Production ceph.conf Example

```ini
[global]
# Cluster identity
fsid = a7f64266-0894-4f1e-a635-d0aeaca0e993
mon_initial_members = mon-a, mon-b, mon-c
mon_host = 10.0.0.1, 10.0.0.2, 10.0.0.3

# Authentication
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

# Network
public_network = 10.0.0.0/24
cluster_network = 10.1.0.0/24

# Logging (minimal for production)
debug_ms = 0/1
debug_osd = 0/1
debug_mon = 0/1

# Capacity thresholds
mon_osd_full_ratio = 0.95
mon_osd_nearfull_ratio = 0.85
mon_osd_backfillfull_ratio = 0.90

[osd]
# BlueStore (default backend)
osd_objectstore = bluestore
bluestore_cache_autotune = true
bluestore_cache_size_hdd = 1073741824
bluestore_cache_size_ssd = 3221225472

# Scrubbing - limit to off-peak hours
osd_scrub_begin_hour = 22
osd_scrub_end_hour = 6
osd_scrub_sleep = 0.1
osd_deep_scrub_interval = 604800

# Recovery - conservative settings for production
osd_recovery_max_active = 3
osd_max_backfills = 1
osd_recovery_op_priority = 3

# OSD heartbeat
osd_heartbeat_interval = 6
osd_heartbeat_grace = 20

[mon]
# Monitor tuning
mon_allow_pool_delete = false
mon_osd_min_in_ratio = 0.75

[client]
# RBD cache (for kernel clients)
rbd_cache = true
rbd_cache_size = 134217728
rbd_cache_max_dirty = 100663296
rbd_cache_target_dirty = 67108864
```

## Translating to Rook CephCluster

In Rook, apply these settings through the CR:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephConfig:
    global:
      mon_osd_full_ratio: "0.95"
      mon_osd_nearfull_ratio: "0.85"
    osd:
      osd_scrub_begin_hour: "22"
      osd_scrub_end_hour: "6"
      osd_recovery_max_active: "3"
    mon:
      mon_allow_pool_delete: "false"
```

## Important Production Considerations

- Always set separate `public_network` and `cluster_network` to isolate replication traffic
- Keep debug levels at 0/1 in production to avoid excessive log volume
- Set `mon_allow_pool_delete = false` to prevent accidental pool deletion
- Tune BlueStore cache sizes based on available RAM per OSD node

## Summary

A production `ceph.conf` should configure network separation, conservative recovery settings, off-peak scrubbing, and appropriate capacity thresholds. In Rook, translate these settings into the `cephConfig` field of your `CephCluster` CR. Keep the file minimal and rely on the monitor configuration database for runtime tuning.
