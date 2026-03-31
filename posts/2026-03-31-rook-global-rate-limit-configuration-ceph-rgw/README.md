# How to Enable Global Rate Limit Configuration in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Rate Limit, Administration, Object Storage, Global, QoS

Description: Configure global rate limits in Ceph RGW to apply default request throttling to all users and buckets cluster-wide without setting per-user rules.

---

While per-user and per-bucket rate limits provide fine-grained control, Ceph RGW also supports global rate limits that apply as defaults across all users and buckets. Global limits provide a safety net to prevent any single entity from overwhelming the gateway without requiring individual configuration.

## Global Rate Limit Scope

Global rate limits act as catch-all defaults. When a user or bucket has no specific rate limit configured, the global limit applies. When a specific limit is set, it overrides the global default.

## Setting Global Rate Limits

Global rate limits are configured at the zone level:

```bash
radosgw-admin global quota set \
  --quota-scope user \
  --max-size 107374182400 \
  --max-objects 10000000
```

For global bucket quotas:

```bash
radosgw-admin global quota set \
  --quota-scope bucket \
  --max-size 21474836480 \
  --max-objects 2000000
```

Enable global quotas:

```bash
radosgw-admin global quota enable \
  --quota-scope user

radosgw-admin global quota enable \
  --quota-scope bucket
```

## Setting Global Rate Limits via Config

For rate limiting (ops and bytes), set global defaults via Ceph config:

```bash
ceph config set client.rgw rgw_max_listing_results 1000
ceph config set client.rgw rgw_max_concurrent_requests 1024
```

Or via the global ratelimit command (Quincy and later):

```bash
radosgw-admin global ratelimit set \
  --ratelimit-scope user \
  --max-read-ops 5000 \
  --max-write-ops 2000 \
  --max-read-bytes 524288000 \
  --max-write-bytes 209715200

radosgw-admin global ratelimit enable \
  --ratelimit-scope user
```

## Viewing Global Rate Limit Settings

```bash
radosgw-admin global ratelimit get \
  --ratelimit-scope user
```

Output:

```json
{
  "enabled": true,
  "max_read_ops": 5000,
  "max_write_ops": 2000,
  "max_read_bytes": 524288000,
  "max_write_bytes": 209715200
}
```

## Global Quota vs Global Rate Limit

| Feature | Global Quota | Global Rate Limit |
|---------|-------------|-------------------|
| Controls | Storage size and object count | Ops/min and bytes/min |
| Enforcement | At write time (storage check) | Per time window |
| Override | Per-user or per-bucket quota | Per-user or per-bucket ratelimit |

## Disabling Global Rate Limits

```bash
radosgw-admin global ratelimit disable \
  --ratelimit-scope user

radosgw-admin global ratelimit disable \
  --ratelimit-scope bucket
```

## Priority of Rate Limits

The priority order for rate limit resolution:

1. Per-bucket rate limit (highest priority)
2. Per-user rate limit
3. Global rate limit (default)

This means per-bucket and per-user settings always take precedence over the global default. If neither is set, the global limit applies.

## Rook: Applying Global Config

In Rook, apply global rate limit changes via the toolbox:

```bash
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin global ratelimit set \
  --ratelimit-scope user \
  --max-read-ops 5000 \
  --max-write-ops 2000 && \
  radosgw-admin global ratelimit enable --ratelimit-scope user
```

## Summary

Global rate limits in Ceph RGW provide cluster-wide defaults that apply to all users and buckets without individual configuration. They act as a safety net below per-user and per-bucket limits in the priority hierarchy. Use global limits as the baseline for your cluster and override them with per-user or per-bucket settings for special cases.
