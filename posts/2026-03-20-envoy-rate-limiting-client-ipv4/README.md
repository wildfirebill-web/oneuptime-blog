# How to Configure Envoy Rate Limiting Based on Client IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Envoy, Rate Limiting, IPv4, Security, Filter, Service Mesh

Description: Implement per-IPv4 rate limiting in Envoy using the local rate limit filter and remote rate limit service to protect upstream services from traffic abuse.

## Introduction

Envoy supports two rate limiting modes: local (per-instance, no external service) and global (using an external rate limit service like Lyft's ratelimit). Local rate limiting is simpler to deploy and suitable for most use cases.

## Local Rate Limiting by Client IP

```yaml
# /etc/envoy/envoy.yaml

static_resources:
  listeners:
    - name: http_listener
      address:
        socket_address: { address: 0.0.0.0, port_value: 8080 }
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/" }
                          route: { cluster: backend_cluster }
                          # Per-route rate limit descriptors
                          rate_limits:
                            - actions:
                                - remote_address: {}   # Use client IP as key
                http_filters:
                  # Local rate limit filter (runs in Envoy, no external service)
                  - name: envoy.filters.http.local_ratelimit
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
                      stat_prefix: http_local_rate_limiter

                      # Token bucket: 100 requests per second
                      token_bucket:
                        max_tokens: 100
                        tokens_per_fill: 100
                        fill_interval: 1s

                      filter_enabled:
                        default_value:
                          numerator: 100
                          denominator: HUNDRED

                      filter_enforced:
                        default_value:
                          numerator: 100
                          denominator: HUNDRED

                      # Return 429 when rate limited
                      status:
                        code: 429

                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

## Per-Route Rate Limits with Remote Rate Limit Service

For per-IP granular limits using an external ratelimit service:

```yaml
http_filters:
  - name: envoy.filters.http.ratelimit
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.ratelimit.v3.RateLimit
      domain: production
      request_type: external
      stage: 0
      # Connect to the ratelimit service
      rate_limit_service:
        grpc_service:
          envoy_grpc:
            cluster_name: rate_limit_cluster
          timeout: 0.25s
        transport_api_version: V3
```

Route configuration with descriptors:

```yaml
routes:
  - match: { prefix: "/api/" }
    route: { cluster: api_cluster }
    rate_limits:
      - actions:
          - remote_address: {}     # Rate limit key: client IP
      - actions:
          - header_value_match:
              descriptor_value: api_endpoint
              headers:
                - name: ":path"
                  prefix_match: "/api/"
```

## Response Headers for Rate Limit Info

```yaml
# Add rate limit headers to responses

response_headers_to_add:
  - header:
      key: "X-RateLimit-Limit"
      value: "100"
  - header:
      key: "Retry-After"
      value: "1"
    keep_empty_value: false
```

## Monitoring Rate Limiting

```bash
# View rate limit stats
curl http://127.0.0.1:9901/stats | grep rate_limit

# Key metrics:
# http.ingress_http.local_rate_limit.enabled    → requests checked
# http.ingress_http.local_rate_limit.enforced   → rate limit applied
# http.ingress_http.local_rate_limit.ok         → requests passed
# http.ingress_http.local_rate_limit.rate_limited → requests dropped

# Real-time monitoring
watch -n2 "curl -s http://127.0.0.1:9901/stats | grep local_rate_limit"
```

## Conclusion

Envoy local rate limiting via the `envoy.filters.http.local_ratelimit` filter provides per-instance request throttling without external dependencies. Each Envoy instance enforces its own bucket, making it suitable for even traffic distributions. For globally coordinated per-IP limits across multiple Envoy instances, integrate with the Lyft ratelimit service using `envoy.filters.http.ratelimit` with `remote_address` descriptors.
