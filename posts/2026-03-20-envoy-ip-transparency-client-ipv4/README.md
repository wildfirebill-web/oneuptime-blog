# How to Configure Envoy IP Transparency to Preserve Client IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Envoy, IP Transparency, IPv4, Client IP, PROXY Protocol, Service Mesh

Description: Configure Envoy to preserve original client IPv4 addresses using IP transparency, XFF headers, or the PROXY protocol when forwarding traffic to upstream services.

## Introduction

Without IP transparency, backends see Envoy's IP address rather than the real client IP. Envoy provides multiple mechanisms to preserve client IPv4 addresses depending on the protocol and backend capabilities.

## Method 1: X-Forwarded-For Headers

The standard HTTP approach adds `X-Forwarded-For` to requests:

```yaml
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

                # Configure how XFF is handled
                use_remote_address: true        # Use real client IP for XFF
                xff_num_trusted_hops: 0         # 0 = Envoy is the edge proxy

                # Skip XFF for internal traffic if needed
                # skip_xff_append: false        # Default: append client IP

                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/" }
                          route: { cluster: backend_cluster }

                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

## Method 2: PROXY Protocol to Upstream

Send PROXY protocol headers to backends:

```yaml
clusters:
  - name: backend_cluster
    type: STATIC
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: backend_cluster
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address: { address: 192.168.1.10, port_value: 8080 }

    # Send PROXY protocol v1 or v2 to backend
    upstream_bind_config: {}
    transport_socket:
      name: envoy.transport_sockets.upstream_http_11
      # For PROXY protocol:
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        upstream_proxy_protocol_config:
          version: V2   # PROXY protocol version 2 (binary)
```

## Method 3: Accepting PROXY Protocol from Upstream

When Envoy sits behind an upstream load balancer sending PROXY protocol:

```yaml
listeners:
  - name: proxy_protocol_listener
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    listener_filters:
      # Read PROXY protocol header before passing to filter chain
      - name: envoy.filters.listener.proxy_protocol
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.listener.proxy_protocol.v3.ProxyProtocol
    filter_chains:
      - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: ingress_http
              use_remote_address: true
              route_config:
                name: local_route
                virtual_hosts:
                  - name: backend
                    domains: ["*"]
                    routes:
                      - match: { prefix: "/" }
                        route: { cluster: backend_cluster }
              http_filters:
                - name: envoy.filters.http.router
                  typed_config:
                    "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

## Verifying Client IP Forwarding

```bash
# Create a debug backend that echoes headers
python3 -c "
from http.server import HTTPServer, BaseHTTPRequestHandler
import json

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-Type', 'application/json')
        self.end_headers()
        headers = dict(self.headers)
        self.wfile.write(json.dumps(headers).encode())

HTTPServer(('0.0.0.0', 8080), Handler).serve_forever()
" &

# Send request through Envoy
curl http://localhost:8080/ | jq '."X-Forwarded-For"'
# Should show the real client IP
```

## Conclusion

Envoy IP transparency can be achieved via `X-Forwarded-For` (HTTP, set `use_remote_address: true`), PROXY protocol to backends (for TCP services), or accepting PROXY protocol from upstream load balancers via the `proxy_protocol` listener filter. Choose XFF for HTTP workloads, PROXY protocol for TCP proxying where HTTP headers are unavailable.
