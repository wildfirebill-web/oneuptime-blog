# How to Set Up Envoy with IPv4 and IPv6 Dual-Stack Listeners

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Envoy, IPv4, IPv6, Dual Stack, Listeners, Networking, Configuration

Description: Learn how to configure Envoy listeners that accept both IPv4 and IPv6 connections to support dual-stack network deployments.

---

Dual-stack deployments serve both IPv4 and IPv6 clients simultaneously. Envoy supports this by binding listeners to `0.0.0.0` (IPv4), `::` (IPv6), or by using `::` with IPv4-mapped addresses on Linux.

## Option 1: Separate Listeners per Protocol

The cleanest approach - one listener for IPv4, one for IPv6.

```yaml
# envoy-config.yaml

static_resources:
  listeners:
    # IPv4 listener
    - name: http_ipv4
      address:
        socket_address:
          address: 0.0.0.0      # All IPv4 interfaces
          port_value: 8080
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ipv4_ingress
                route_config:
                  virtual_hosts:
                    - name: app
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/" }
                          route: { cluster: backend }
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

    # IPv6 listener
    - name: http_ipv6
      address:
        socket_address:
          address: "::"          # All IPv6 interfaces
          port_value: 8080
          ipv4_compat: false     # Pure IPv6 only
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ipv6_ingress
                route_config:
                  virtual_hosts:
                    - name: app
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/" }
                          route: { cluster: backend }
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

## Option 2: Single Listener with ipv4_compat

On Linux, an IPv6 socket can accept IPv4 clients via IPv4-mapped IPv6 addresses when `ipv4_compat: true` is set.

```yaml
listeners:
  - name: dual_stack_listener
    address:
      socket_address:
        address: "::"
        port_value: 8080
        # Allow IPv4 clients to connect via ::ffff:x.x.x.x mapping
        ipv4_compat: true
    filter_chains:
      - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: dual_stack
              use_remote_address: true   # Correctly extract IPv4 from mapped address
              route_config:
                virtual_hosts:
                  - name: app
                    domains: ["*"]
                    routes:
                      - match: { prefix: "/" }
                        route: { cluster: backend }
              http_filters:
                - name: envoy.filters.http.router
                  typed_config:
                    "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

## Backends: IPv4 Upstream from a Dual-Stack Listener

Regardless of which protocol the client uses, the upstream cluster uses IPv4.

```yaml
clusters:
  - name: backend
    type: STATIC
    connect_timeout: 5s
    load_assignment:
      cluster_name: backend
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 10.0.0.10   # IPv4 backend
                    port_value: 8080
```

## Verifying Dual-Stack Listening

```bash
# Check Envoy is listening on both IPv4 and IPv6
ss -tlnp | grep envoy

# Test IPv4 connection
curl -4 http://127.0.0.1:8080/

# Test IPv6 connection
curl -6 http://[::1]:8080/
```

## Key Takeaways

- Separate listeners for IPv4 (`0.0.0.0`) and IPv6 (`::`) is the clearest dual-stack approach.
- `ipv4_compat: true` on an IPv6 listener accepts IPv4 clients via mapped addresses on Linux.
- Set `use_remote_address: true` to correctly log the IPv4 address of clients connecting via mapped IPv6.
- Upstream clusters remain IPv4-only - Envoy handles protocol translation between listener and cluster.
