# How to Configure gRPC with Envoy Proxy for IPv4 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: gRPC, Envoy, IPv4, Proxy, Networking, Kubernetes

Description: Learn how to configure Envoy Proxy as a gRPC proxy for IPv4 traffic, including HTTP/2 routing, load balancing, circuit breaking, and health checking configuration.

## Why Envoy for gRPC?

Envoy understands gRPC natively: it decodes gRPC status codes, collects per-method metrics, supports retries and circuit breakers at the RPC level, and is the standard data plane for service meshes (Istio, AWS App Mesh).

## Basic Envoy Configuration for gRPC

```yaml
# envoy.yaml
static_resources:
  listeners:
    - name: grpc_listener
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 50051

      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: grpc_proxy
                codec_type: AUTO
                route_config:
                  name: grpc_routes
                  virtual_hosts:
                    - name: grpc_backends
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/helloworld.Greeter"
                          route:
                            cluster: greeter_cluster
                            timeout: 10s
                            retry_policy:
                              retry_on: "cancelled,deadline-exceeded,resource-exhausted"
                              num_retries: 3

                http_filters:
                  - name: envoy.filters.http.grpc_stats
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_stats.v3.FilterConfig
                      emit_filter_state: true
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: greeter_cluster
      connect_timeout: 5s
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      http2_protocol_options: {}    # gRPC requires HTTP/2 to backends
      load_assignment:
        cluster_name: greeter_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: 10.0.0.1
                      port_value: 50051
              - endpoint:
                  address:
                    socket_address:
                      address: 10.0.0.2
                      port_value: 50051
      health_checks:
        - timeout: 2s
          interval: 10s
          grpc_health_check: {}     # use gRPC Health protocol

admin:
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9901
```

## Circuit Breaker Configuration

```yaml
clusters:
  - name: greeter_cluster
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          max_connections:     100
          max_pending_requests: 100
          max_requests:        1000
          max_retries:         3
```

## Docker Compose: Envoy in Front of gRPC Service

```yaml
services:
  envoy:
    image: envoyproxy/envoy:v1.28-latest
    volumes:
      - ./envoy.yaml:/etc/envoy/envoy.yaml
    ports:
      - "50051:50051"
      - "9901:9901"

  greeter:
    image: myrepo/greeter:latest
    expose:
      - "50051"
```

## Viewing gRPC Metrics

```bash
# Envoy admin API — gRPC stats per method
curl http://localhost:9901/stats | grep grpc

# Example metrics:
# cluster.greeter_cluster.grpc.helloworld.Greeter.SayHello.success: 1000
# cluster.greeter_cluster.grpc.helloworld.Greeter.SayHello.failure: 2
```

## Conclusion

Envoy's `http_connection_manager` filter with `http2_protocol_options` on the cluster enables gRPC proxying with automatic retry, circuit breaking, and per-method metrics via the `grpc_stats` filter. Use `STRICT_DNS` cluster type with multiple endpoints for load balancing, and enable `grpc_health_check` in the health_checks section to use the gRPC Health protocol instead of a simple TCP connect. In production, Envoy is typically managed by a service mesh control plane (Istio, Consul Connect) rather than static configuration files.
