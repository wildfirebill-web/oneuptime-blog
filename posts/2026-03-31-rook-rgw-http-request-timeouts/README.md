# How to Set HTTP Request Timeouts in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, HTTP, Timeout, Performance

Description: Configure HTTP request timeout parameters in Ceph RGW to handle slow clients, prevent resource exhaustion, and improve gateway reliability.

---

Ceph RGW accepts HTTP connections from many concurrent clients. Timeouts prevent slow or unresponsive clients from consuming resources indefinitely.

## Key HTTP Timeout Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `rgw_request_timeout_msec` | 60000 (60s) | Total request processing timeout |
| `rgw_curl_low_speed_time` | 60 | Seconds before aborting a slow CURL connection |
| `rgw_curl_low_speed_limit` | 1024 | Minimum bytes/sec to not be considered slow |

## Checking Current Timeout Settings

```bash
ceph config get client.rgw rgw_request_timeout_msec
ceph config get client.rgw rgw_curl_low_speed_time
ceph config get client.rgw rgw_curl_low_speed_limit
```

## Configuring Request Timeouts

```bash
# Set overall request timeout to 120 seconds
ceph config set client.rgw rgw_request_timeout_msec 120000

# For large object uploads, increase significantly
ceph config set client.rgw rgw_request_timeout_msec 600000
```

## Configuring Low-Speed Timeouts

Low-speed timeouts abort connections that transfer data too slowly:

```bash
# Abort if speed drops below 4KB/s for 60 seconds
ceph config set client.rgw rgw_curl_low_speed_limit 4096
ceph config set client.rgw rgw_curl_low_speed_time 60
```

## Frontend-Level Timeouts

The beast frontend also supports connection-level timeouts:

```bash
ceph config set client.rgw rgw_frontends \
  "beast port=7480 tcp_nodelay=1"
```

For idle connection management, configure at the OS level:

```bash
# Set TCP keepalive (in seconds) on the RGW host
sysctl -w net.ipv4.tcp_keepalive_time=120
sysctl -w net.ipv4.tcp_keepalive_intvl=30
sysctl -w net.ipv4.tcp_keepalive_probes=5
```

In Kubernetes, configure these via a DaemonSet or init container.

## Applying in Rook

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [client.rgw.my-store.a]
    rgw_request_timeout_msec = 120000
    rgw_curl_low_speed_limit = 4096
    rgw_curl_low_speed_time = 60
```

Apply and restart:

```bash
kubectl apply -f rook-config-override.yaml
kubectl -n rook-ceph rollout restart deployment rook-ceph-rgw-my-store-a
```

## Diagnosing Timeout Issues

```bash
# Check for timeout-related log messages
kubectl -n rook-ceph logs deploy/rook-ceph-rgw-my-store-a --tail=200 | \
  grep -iE "timeout|timed out|slow"

# Monitor active connections
kubectl -n rook-ceph exec -it deploy/rook-ceph-rgw-my-store-a -- \
  ss -tnp | grep ESTABLISHED | wc -l
```

## Summary

HTTP timeout parameters in Ceph RGW protect against slow clients and resource exhaustion. Use `rgw_request_timeout_msec` for overall request limits, and `rgw_curl_low_speed_limit`/`rgw_curl_low_speed_time` for slow connection detection. Increase timeouts for large object upload workloads and apply settings via Rook's config override ConfigMap.
