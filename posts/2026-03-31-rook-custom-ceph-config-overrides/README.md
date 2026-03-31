# How to Pass Custom Ceph Config Overrides in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Configuration, Override, Tuning

Description: Override individual Ceph configuration parameters in Rook using the CephCluster configOverrides section to tune OSD memory, network timeouts, and other advanced settings.

---

## Why You Need Config Overrides

Rook configures Ceph with sensible defaults, but advanced deployments often need to tune specific parameters that are not exposed as first-class CRD fields. The `cephConfig` section in `CephCluster` lets you inject arbitrary Ceph configuration key-value pairs.

Common use cases include:
- Reducing OSD memory usage on nodes with limited RAM
- Tuning RocksDB and BlueStore cache settings
- Adjusting network timeouts for high-latency links
- Enabling experimental features for testing

## Setting Config Overrides via CephCluster

Add the `cephConfig` section directly to the `CephCluster` spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephConfig:
    global:
      osd_pool_default_size: "3"
      osd_pool_default_min_size: "2"
      mon_warn_on_insecure_global_id_reclaim_allowed: "false"
    osd:
      osd_memory_target: "4294967296"    # 4 GiB per OSD
      bluestore_cache_size: "1073741824" # 1 GiB BlueStore cache
    mon:
      mon_osd_down_out_interval: "600"
    mgr:
      mgr_stats_period: "5"
```

Keys use underscores (Ceph internal format), not dots. Rook writes these to the Ceph configuration database (`ceph config set`) rather than to a flat file.

## Setting Config on Individual Daemon Types

The section headers (`global`, `osd`, `mon`, `mgr`, `mds`, `client`) map to Ceph config sections. Settings under `global` apply to all daemons; type-specific sections override global for that daemon type.

```yaml
cephConfig:
  global:
    auth_cluster_required: "cephx"
    auth_service_required: "cephx"
  osd:
    osd_scrub_min_interval: "86400"    # minimum 1 day between scrubs
    osd_scrub_max_interval: "604800"   # maximum 7 days between scrubs
    osd_deep_scrub_interval: "604800"  # deep scrub weekly
```

## Tuning Memory Limits

For nodes with constrained RAM, reduce OSD memory targets:

```yaml
cephConfig:
  osd:
    osd_memory_target: "2147483648"   # 2 GiB
    osd_memory_base: "268435456"      # 256 MiB base
    osd_memory_cache_min: "134217728" # 128 MiB minimum cache
```

Reducing below 2 GiB per OSD risks degraded performance but can be necessary on small clusters.

## Verifying Applied Configuration

After the cluster reconciles, verify the settings are active:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config dump
```

Query a specific key:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config get osd osd_memory_target
```

## Temporarily Overriding Without CRD Changes

For quick testing, apply config changes directly through the toolbox without modifying the CRD:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_recovery_max_active 3
```

Note: Rook will overwrite manual changes during the next reconciliation if the same key is set in the CRD. Use `cephConfig` in the CRD for persistent changes.

## Summary

The `cephConfig` override mechanism in Rook provides a structured way to tune any Ceph parameter without modifying configuration files on individual nodes. Use it to adjust memory limits, scrub schedules, and network timeouts. Always verify changes via `ceph config dump` and prefer CRD-based overrides over manual toolbox changes to ensure settings survive operator reconciliation.
