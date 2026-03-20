# How to Set Envoy DNS Resolution to V4_ONLY Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Envoy, DNS, IPv4, V4_ONLY, Service Mesh, Networking

Description: Configure Envoy to resolve hostnames to IPv4 addresses only using the V4_ONLY dns_lookup_family setting, preventing IPv6 resolution on IPv4-only networks.

## Introduction

By default, Envoy uses AUTO DNS resolution, which prefers IPv6 when both A and AAAA records exist. Setting `dns_lookup_family: V4_ONLY` forces all hostname resolution to return only IPv4 (A record) results, essential for IPv4-only backend networks.

## Setting V4_ONLY on a Cluster

```yaml
# /etc/envoy/envoy.yaml

static_resources:
  clusters:
    - name: backend_service
      connect_timeout: 5s
      # STRICT_DNS: resolve hostname, get all A records
      type: STRICT_DNS

      # V4_ONLY: only resolve A records, ignore AAAA
      dns_lookup_family: V4_ONLY

      lb_policy: ROUND_ROBIN

      load_assignment:
        cluster_name: backend_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      # Hostname resolved to IPv4 only
                      address: api.service.internal
                      port_value: 8080
```

## DNS Lookup Families Compared

| Setting | Behavior |
|---|---|
| `AUTO` | Try IPv6 first, fall back to IPv4 |
| `V4_ONLY` | Only resolve A records (IPv4) |
| `V6_ONLY` | Only resolve AAAA records (IPv6) |
| `V4_PREFERRED` | Prefer IPv4, fall back to IPv6 |

## Applying V4_ONLY Across Multiple Clusters

```yaml
static_resources:
  clusters:
    - name: user_service
      connect_timeout: 5s
      type: STRICT_DNS
      dns_lookup_family: V4_ONLY   # IPv4 only
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: user_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: users.internal.svc
                      port_value: 8080

    - name: order_service
      connect_timeout: 5s
      type: STRICT_DNS
      dns_lookup_family: V4_ONLY   # IPv4 only
      lb_policy: LEAST_REQUEST
      load_assignment:
        cluster_name: order_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: orders.internal.svc
                      port_value: 8080
```

## DNS Refresh Rate

Control how often Envoy re-resolves hostnames:

```yaml
clusters:
  - name: backend_service
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    # Re-resolve DNS every 5 seconds
    dns_refresh_rate: 5s
    # Minimum TTL override (uses max of DNS TTL and this value)
    dns_min_refresh_rate: 5s
    load_assignment:
      cluster_name: backend_service
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: dynamic-service.internal
                    port_value: 8080
```

## Verifying DNS Resolution

```bash
# Check which IPs Envoy resolved for a cluster
curl http://127.0.0.1:9901/clusters | grep backend_service

# Sample output shows IPv4 addresses:
# backend_service::192.168.1.10:8080::cx_active::0
# backend_service::192.168.1.11:8080::cx_active::0

# Verify no IPv6 endpoints
curl http://127.0.0.1:9901/clusters | grep backend_service | grep -v '^\:\:'
```

## Conclusion

Setting `dns_lookup_family: V4_ONLY` on Envoy clusters ensures all DNS resolution returns IPv4 addresses, preventing connection failures on IPv4-only backend networks. Use it with `STRICT_DNS` or `LOGICAL_DNS` cluster types for hostname-based endpoints, and tune `dns_refresh_rate` for appropriate responsiveness to DNS changes in dynamic environments.
