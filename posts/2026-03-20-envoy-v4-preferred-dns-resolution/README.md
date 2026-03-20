# How to Use V4_PREFERRED DNS Resolution Policy in Envoy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Envoy, DNS, IPv4, IPv6, Configuration, Service Mesh, Networking

Description: Learn how to configure Envoy's V4_PREFERRED DNS lookup family to ensure upstream clusters resolve to IPv4 addresses on dual-stack networks.

---

On dual-stack networks, DNS queries return both A (IPv4) and AAAA (IPv6) records. Envoy's `dns_lookup_family` field in cluster configuration controls which address family is preferred or required for upstream resolution.

## DNS Lookup Family Options

| Value | Behavior |
|-------|----------|
| `AUTO` | Use IPv6 if available, fall back to IPv4 |
| `V4_ONLY` | Only use A records (IPv4) |
| `V6_ONLY` | Only use AAAA records (IPv6) |
| `V4_PREFERRED` | Prefer A records; fall back to AAAA if no A records exist |
| `ALL` | Return all addresses (both A and AAAA) |

## Setting V4_PREFERRED on a Cluster

The `dns_lookup_family` field is set in the cluster's configuration.

```yaml
# envoy-config.yaml
static_resources:
  clusters:
    - name: my_backend
      type: STRICT_DNS       # Envoy performs DNS resolution for this cluster
      dns_lookup_family: V4_PREFERRED   # Prefer IPv4; fall back to IPv6 if needed
      connect_timeout: 5s
      load_assignment:
        cluster_name: my_backend
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: backend.internal  # Hostname to resolve
                      port_value: 8080
```

## V4_ONLY for Strictly IPv4 Backends

When your backend only has an A record and you want to fail fast if DNS returns AAAA:

```yaml
clusters:
  - name: ipv4_only_service
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    connect_timeout: 5s
    load_assignment:
      cluster_name: ipv4_only_service
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: legacy-app.internal
                    port_value: 3000
```

## Minimal Full Envoy Config with V4_PREFERRED

```yaml
# envoy-config.yaml
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 10000
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: local_service
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/" }
                          route: { cluster: my_backend }
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: my_backend
      type: STRICT_DNS
      dns_lookup_family: V4_PREFERRED
      connect_timeout: 5s
      load_assignment:
        cluster_name: my_backend
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: backend.internal
                      port_value: 8080

admin:
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9901
```

## Verifying Resolution

```bash
# Check the current cluster endpoints and their resolved IP addresses
curl -s http://localhost:9901/clusters | grep my_backend

# View DNS resolution stats
curl -s http://localhost:9901/stats | grep dns
```

## Key Takeaways

- Set `dns_lookup_family: V4_PREFERRED` in the cluster to prefer IPv4 on dual-stack DNS.
- Use `V4_ONLY` when the backend is guaranteed to be IPv4-only.
- `STRICT_DNS` clusters re-resolve the hostname periodically; monitor the admin endpoint for the current resolved addresses.
- Check `curl localhost:9901/clusters` to verify which IP addresses Envoy is using.
