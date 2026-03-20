# How to Configure Envoy HTTP Connection Manager for IPv4 Listeners

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Envoy, HTTP Connection Manager, IPv4, Listeners, Proxy, Configuration, Service Mesh

Description: Learn how to configure Envoy's HTTP Connection Manager filter on IPv4 listeners to handle routing, headers, and access logging for HTTP traffic.

---

The HTTP Connection Manager (HCM) is Envoy's primary filter for HTTP traffic. It handles request routing, header manipulation, access logging, and HTTP protocol negotiation. Understanding its configuration is essential for any Envoy-based proxy or service mesh setup.

## Basic IPv4 Listener with HCM

```yaml
# envoy-config.yaml

static_resources:
  listeners:
    - name: http_listener
      address:
        socket_address:
          # Bind to all IPv4 interfaces on port 80
          address: 0.0.0.0
          port_value: 80
          # protocol: TCP is the default; explicitly set for clarity
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager

                # Prefix used in stats (metrics) for this HCM instance
                stat_prefix: ingress_http

                # Generate a unique request ID for tracing
                generate_request_id: true

                # Forward the X-Forwarded-For header with the client's IP
                use_remote_address: true

                # Route configuration
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: app_service
                      domains: ["*"]          # Match all Host headers
                      routes:
                        - match:
                            prefix: "/api/"   # Route /api/* to the API backend
                          route:
                            cluster: api_cluster
                        - match:
                            prefix: "/"       # Default route
                          route:
                            cluster: web_cluster

                # Filter chain: router must be last
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: api_cluster
      type: STATIC
      connect_timeout: 5s
      load_assignment:
        cluster_name: api_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address: { address: 10.0.0.10, port_value: 8080 }

    - name: web_cluster
      type: STATIC
      connect_timeout: 5s
      load_assignment:
        cluster_name: web_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address: { address: 10.0.0.20, port_value: 80 }

admin:
  address:
    socket_address: { address: 127.0.0.1, port_value: 9901 }
```

## Adding Access Logging

```yaml
http_connection_manager:
  stat_prefix: ingress_http
  access_log:
    - name: envoy.access_loggers.stdout
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
        log_format:
          text_format_source:
            inline_string: "[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %BYTES_SENT% \"%DOWNSTREAM_REMOTE_ADDRESS%\"\n"
```

## Enabling HTTP/2 Upgrade

```yaml
http_connection_manager:
  stat_prefix: ingress_http
  codec_type: AUTO          # Auto-detect HTTP/1.1 or HTTP/2
  use_remote_address: true  # Populate X-Forwarded-For with client IPv4
  route_config: { ... }
  http_filters: [...]
```

## Header Manipulation

```yaml
virtual_hosts:
  - name: app
    domains: ["*"]
    routes:
      - match: { prefix: "/" }
        route: { cluster: backend }
        # Add custom headers to requests forwarded to the backend
    request_headers_to_add:
      - header:
          key: X-Source-IP
          value: "%DOWNSTREAM_REMOTE_ADDRESS_WITHOUT_PORT%"
        keep_empty_value: false
```

## Key Takeaways

- Bind the listener `address` to `0.0.0.0` for all IPv4 interfaces or a specific IP.
- Set `use_remote_address: true` to correctly populate `X-Forwarded-For` with the client's IPv4.
- The router filter (`envoy.filters.http.router`) must be the last HTTP filter in the chain.
- Use `stat_prefix` to namespace metrics for multiple HCM instances.
