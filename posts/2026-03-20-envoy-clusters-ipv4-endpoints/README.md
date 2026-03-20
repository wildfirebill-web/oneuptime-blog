# How to Configure Envoy Clusters with IPv4 Upstream Endpoints

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Envoy, Clusters, IPv4, Endpoints, Load Balancing, Upstream

Description: Configure Envoy clusters to define IPv4 upstream endpoints with load balancing, health checks, and connection pool settings for HTTP and TCP traffic.

## Introduction

In Envoy, a "cluster" defines a group of upstream servers (endpoints) that receive proxied requests. Each endpoint is an IPv4 address:port pair. Clusters control load balancing policy, health checking, TLS settings, and connection pool behavior.

## Static Cluster with IPv4 Endpoints

```yaml
# /etc/envoy/envoy.yaml

static_resources:
  clusters:
    - name: my_service
      # connect_timeout: max time to establish TCP connection
      connect_timeout: 5s

      # STATIC: use hardcoded IPv4 endpoints
      type: STATIC

      # Load balancing policy
      lb_policy: ROUND_ROBIN

      load_assignment:
        cluster_name: my_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: 192.168.1.10
                      port_value: 8080
              - endpoint:
                  address:
                    socket_address:
                      address: 192.168.1.11
                      port_value: 8080
              - endpoint:
                  address:
                    socket_address:
                      address: 192.168.1.12
                      port_value: 8080
```

## Cluster with HTTP/2 and Connection Pooling

```yaml
clusters:
  - name: grpc_service
    connect_timeout: 5s
    type: STATIC
    lb_policy: LEAST_REQUEST

    # HTTP/2 protocol for gRPC backends
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}

    # Connection pool tuning
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          max_connections: 1024
          max_pending_requests: 1024
          max_requests: 1024
          max_retries: 3

    load_assignment:
      cluster_name: grpc_service
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 192.168.2.10
                    port_value: 9090
```

## DNS-Based Cluster (STRICT_DNS)

Resolve hostnames to IPv4 addresses at runtime:

```yaml
clusters:
  - name: dynamic_service
    connect_timeout: 5s
    # STRICT_DNS: continuously resolve hostname, use ALL returned A records
    type: STRICT_DNS

    # Prefer IPv4 over IPv6 (dns_lookup_family)
    dns_lookup_family: V4_ONLY

    lb_policy: ROUND_ROBIN

    load_assignment:
      cluster_name: dynamic_service
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: my-service.internal   # Resolved to IPv4
                    port_value: 8080
```

## Active Health Checking

```yaml
clusters:
  - name: healthy_service
    connect_timeout: 5s
    type: STATIC
    lb_policy: ROUND_ROBIN

    # HTTP health checks
    health_checks:
      - timeout: 5s
        interval: 10s
        unhealthy_threshold: 3   # Mark unhealthy after 3 failures
        healthy_threshold: 2      # Mark healthy after 2 successes
        http_health_check:
          path: /health
          expected_statuses:
            - start: 200
              end: 299

    load_assignment:
      cluster_name: healthy_service
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 192.168.1.10
                    port_value: 8080
            - endpoint:
                address:
                  socket_address:
                    address: 192.168.1.11
                    port_value: 8080
```

## Verifying Cluster State

```bash
# View cluster endpoint health via Envoy admin API
curl http://127.0.0.1:9901/clusters

# Get detailed cluster stats
curl http://127.0.0.1:9901/stats?filter=cluster.my_service

# Check endpoint health status
curl http://127.0.0.1:9901/clusters | grep -E "my_service.*health_flags"

# List all configured clusters
curl http://127.0.0.1:9901/config_dump | jq '.configs[] | select(.["@type"] | contains("ClustersConfigDump"))'
```

## Conclusion

Envoy clusters define IPv4 upstream endpoints with precise control over load balancing, health checking, and connection behavior. Use `STATIC` type for known IPs, `STRICT_DNS` for hostname-based discovery with V4_ONLY resolution, and configure `circuit_breakers` for connection pool limits. The admin API at `/clusters` provides real-time visibility into endpoint health and traffic statistics.
