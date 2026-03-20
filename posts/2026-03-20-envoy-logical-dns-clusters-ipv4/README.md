# How to Set Up Envoy Logical DNS Clusters for IPv4 Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Envoy, DNS, IPv4, Logical DNS, Cluster, Service Mesh, Configuration

Description: Learn how to configure Envoy LOGICAL_DNS clusters to resolve IPv4 service addresses at connect time for services with frequently changing IPs.

---

Envoy supports several cluster discovery types. `LOGICAL_DNS` resolves the hostname at connection time (not at cluster creation), making it suitable for services like CDNs or external APIs where IPs change frequently and DNS TTLs are short.

## STRICT_DNS vs LOGICAL_DNS

| Feature | STRICT_DNS | LOGICAL_DNS |
|---------|-----------|-------------|
| Resolution timing | Periodic background refresh | At connection time |
| Multiple A records | Creates one endpoint per IP | Uses one IP (first returned) |
| Best for | Internal services with stable IPs | External services, CDNs |
| DNS TTL respected | Yes | Yes |

## Configuring a LOGICAL_DNS Cluster

```yaml
# envoy-config.yaml
static_resources:
  clusters:
    - name: external_api
      # LOGICAL_DNS: resolve at connect time; uses first returned IP
      type: LOGICAL_DNS
      dns_lookup_family: V4_PREFERRED   # Prefer IPv4
      connect_timeout: 5s
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: external_api
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      # Hostname — resolved fresh for each new connection
                      address: api.external-service.com
                      port_value: 443
      # TLS configuration for HTTPS to the external service
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
          sni: api.external-service.com
```

## When to Use LOGICAL_DNS

Use `LOGICAL_DNS` when:
- The upstream has a single IP behind a hostname (e.g., a SaaS API).
- The IP changes frequently and you want DNS to determine the IP per connection.
- You don't want Envoy to cache a list of IPs from round-robin DNS.

## Combining LOGICAL_DNS with Health Checks

```yaml
clusters:
  - name: payment_service
    type: LOGICAL_DNS
    dns_lookup_family: V4_ONLY
    connect_timeout: 3s
    health_checks:
      - timeout: 2s
        interval: 10s
        unhealthy_threshold: 3
        healthy_threshold: 1
        http_health_check:
          path: /health
    load_assignment:
      cluster_name: payment_service
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: payments.internal
                    port_value: 8080
```

## Monitoring DNS Resolution

```bash
# View active upstream hosts in the cluster
curl -s http://localhost:9901/clusters | grep payment_service

# Check DNS-related stats
curl -s http://localhost:9901/stats | grep logical_dns

# Dump the full cluster status
curl -s http://localhost:9901/config_dump | python3 -m json.tool | grep -A 10 external_api
```

## Full Listener + LOGICAL_DNS Example

```yaml
static_resources:
  listeners:
    - name: proxy_listener
      address:
        socket_address: { address: 0.0.0.0, port_value: 8080 }
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: proxy
                route_config:
                  virtual_hosts:
                    - name: api_route
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/" }
                          route: { cluster: external_api }
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: external_api
      type: LOGICAL_DNS
      dns_lookup_family: V4_PREFERRED
      connect_timeout: 5s
      load_assignment:
        cluster_name: external_api
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: api.service.com
                      port_value: 80

admin:
  address:
    socket_address: { address: 127.0.0.1, port_value: 9901 }
```

## Key Takeaways

- `LOGICAL_DNS` resolves the hostname at connect time, ideal for external services with changing IPs.
- Set `dns_lookup_family: V4_PREFERRED` or `V4_ONLY` to ensure IPv4 resolution.
- Unlike `STRICT_DNS`, `LOGICAL_DNS` uses a single resolved IP rather than balancing across all returned IPs.
- Monitor resolution via the Envoy admin API at `/clusters`.
