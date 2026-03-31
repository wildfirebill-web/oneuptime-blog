# How to Set rgwCommandFlags in Rook Object Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Store, Configuration, Kubernetes

Description: Learn how to set rgwCommandFlags in a Rook CephObjectStore to pass custom flags directly to the RADOS Gateway daemon.

---

## What Are rgwCommandFlags

The `rgwCommandFlags` field in a Rook `CephObjectStore` spec allows you to pass arbitrary command-line flags directly to the Ceph RADOS Gateway (RGW) daemon. This gives you fine-grained control over RGW behavior that is not exposed through higher-level Rook configuration options. It maps directly to RGW's own startup flags, letting you tune things like thread counts, cache sizes, and debugging verbosity.

## When to Use rgwCommandFlags

Use `rgwCommandFlags` when you need to:

- Increase the number of RGW frontend worker threads for high-traffic deployments
- Enable or disable specific RGW features at the daemon level
- Set debug log verbosity for troubleshooting
- Pass any RGW flag not otherwise available via the CephObjectStore CRD

## Configuring rgwCommandFlags

Add the `rgwCommandFlags` map under the `gateway` section of your `CephObjectStore` spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
  gateway:
    port: 80
    instances: 1
    rgwCommandFlags:
      rgw_thread_pool_size: "512"
      rgw_max_chunk_size: "4194304"
      debug_rgw: "1"
```

Each key in `rgwCommandFlags` is a Ceph configuration option name (using underscores, not dashes). The value must always be a string, even for numeric settings.

## Common Flags

| Flag | Description |
|------|-------------|
| `rgw_thread_pool_size` | Number of frontend threads per RGW instance |
| `rgw_max_chunk_size` | Maximum size of a single object chunk in bytes |
| `rgw_cache_lru_size` | Number of entries in the RGW LRU cache |
| `debug_rgw` | Debug verbosity level (0-20) |
| `rgw_enable_static_website` | Enable S3 static website hosting |

## Verifying the Flags Are Applied

After applying the updated `CephObjectStore`, restart or wait for the RGW pods to cycle. Then inspect the running process to confirm flags were passed:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-rgw-my-store-a -- \
  ps aux | grep radosgw
```

You can also check the running Ceph configuration values directly:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config show client.rgw.my-store.a
```

## Avoiding Common Mistakes

Flags set via `rgwCommandFlags` override any values in the Ceph config database for that specific daemon. Be careful not to duplicate settings you already manage elsewhere. Always test flag changes in a non-production environment first, as incorrect values can cause RGW to fail to start.

To remove a flag, delete its key from the map and reapply. Rook will regenerate the RGW configuration and restart the daemon.

## Summary

`rgwCommandFlags` is a powerful escape hatch in Rook that lets you pass low-level Ceph RGW options directly to the gateway daemon. By adding key-value pairs under `spec.gateway.rgwCommandFlags` in your `CephObjectStore`, you can tune thread pools, cache sizes, debug levels, and more without waiting for Rook to expose each option as a first-class CRD field.
