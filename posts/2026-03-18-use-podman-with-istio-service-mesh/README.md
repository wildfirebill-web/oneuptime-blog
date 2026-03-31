# How to Use Podman with Istio Service Mesh

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Istio, Service Mesh, Microservice, Networking

Description: Learn how to use Podman with Istio service mesh concepts to implement traffic management, security, and observability for containerized microservices.

---

> Istio service mesh patterns applied to Podman containers bring enterprise-grade traffic management, mutual TLS authentication, and deep observability to your microservices without modifying application code.

Istio is the leading service mesh platform that manages communication between microservices. While Istio is typically deployed on Kubernetes, its core patterns and components can be used with Podman to bring service mesh capabilities to non-Kubernetes container environments. By running Envoy sidecar proxies alongside your Podman containers and using Istio's control plane components, you can implement traffic management, security policies, and comprehensive observability for your containerized services.

---

## Understanding Istio Components

Istio consists of a data plane and a control plane. The data plane is made up of Envoy proxy sidecars deployed alongside each service. The control plane, primarily the `istiod` component, manages and configures these proxies. In a Podman environment, you deploy Envoy sidecars manually as containers within pods, and can use Istio's tools for configuration generation.

## Setting Up the Sidecar Pattern

The fundamental Istio pattern is the sidecar proxy. With Podman pods, this is straightforward:

```bash
# Create a pod for the service

podman pod create \
  --name bookinfo-productpage \
  -p 9080:9080 \
  -p 15000:15000

# Run the application container
podman run -d \
  --pod bookinfo-productpage \
  --name productpage \
  istio/examples-bookinfo-productpage-v1:latest

# Run the Envoy sidecar
podman run -d \
  --pod bookinfo-productpage \
  --name productpage-proxy \
  -v ./envoy/productpage.yaml:/etc/envoy/envoy.yaml:ro,Z \
  envoyproxy/envoy:v1.29-latest
```

The sidecar Envoy configuration:

```yaml
# envoy/productpage.yaml
static_resources:
  listeners:
    - name: inbound
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 9080
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: inbound
                route_config:
                  virtual_hosts:
                    - name: local
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            cluster: local_service
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

    - name: outbound
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 15001
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: outbound
                route_config:
                  virtual_hosts:
                    - name: services
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/reviews"
                          route:
                            cluster: reviews_service
                        - match:
                            prefix: "/details"
                          route:
                            cluster: details_service
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: local_service
      connect_timeout: 5s
      type: STATIC
      load_assignment:
        cluster_name: local_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: 127.0.0.1
                      port_value: 9081

    - name: reviews_service
      connect_timeout: 5s
      type: STRICT_DNS
      load_assignment:
        cluster_name: reviews_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: reviews
                      port_value: 9080

    - name: details_service
      connect_timeout: 5s
      type: STRICT_DNS
      load_assignment:
        cluster_name: details_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: details
                      port_value: 9080

admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 15000
```

## Deploying a Microservice Application

Deploy a complete microservice application with sidecar proxies:

```yaml
# bookinfo-mesh.yml
version: "3"
services:
  productpage:
    image: istio/examples-bookinfo-productpage-v1:latest
    network_mode: "service:productpage-proxy"

  productpage-proxy:
    image: envoyproxy/envoy:v1.29-latest
    ports:
      - "9080:9080"
      - "15000:15000"
    volumes:
      - ./envoy/productpage.yaml:/etc/envoy/envoy.yaml:ro

  reviews-v1:
    image: istio/examples-bookinfo-reviews-v1:latest
    network_mode: "service:reviews-proxy"

  reviews-proxy:
    image: envoyproxy/envoy:v1.29-latest
    volumes:
      - ./envoy/reviews.yaml:/etc/envoy/envoy.yaml:ro

  details:
    image: istio/examples-bookinfo-details-v1:latest
    network_mode: "service:details-proxy"

  details-proxy:
    image: envoyproxy/envoy:v1.29-latest
    volumes:
      - ./envoy/details.yaml:/etc/envoy/envoy.yaml:ro

  ratings:
    image: istio/examples-bookinfo-ratings-v1:latest
    network_mode: "service:ratings-proxy"

  ratings-proxy:
    image: envoyproxy/envoy:v1.29-latest
    volumes:
      - ./envoy/ratings.yaml:/etc/envoy/envoy.yaml:ro
```

## Implementing Mutual TLS

Secure service-to-service communication with mTLS:

```bash
# Generate a CA certificate
openssl req -x509 -newkey rsa:4096 -keyout ca-key.pem -out ca-cert.pem \
  -days 365 -nodes -subj "/CN=Mesh CA"

# Generate a certificate for each service
for service in productpage reviews details ratings; do
    openssl req -newkey rsa:2048 -keyout "${service}-key.pem" -out "${service}-csr.pem" \
      -nodes -subj "/CN=${service}.mesh.local"

    openssl x509 -req -in "${service}-csr.pem" -CA ca-cert.pem -CAkey ca-key.pem \
      -CAcreateserial -out "${service}-cert.pem" -days 365

    rm "${service}-csr.pem"
done
```

Configure Envoy for mTLS:

```yaml
# Add TLS to the cluster configuration
clusters:
  - name: reviews_service
    connect_timeout: 5s
    type: STRICT_DNS
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        common_tls_context:
          tls_certificates:
            - certificate_chain:
                filename: /certs/productpage-cert.pem
              private_key:
                filename: /certs/productpage-key.pem
          validation_context:
            trusted_ca:
              filename: /certs/ca-cert.pem
    load_assignment:
      cluster_name: reviews_service
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: reviews-proxy
                    port_value: 9080
```

## Traffic Management Patterns

Implement common Istio traffic patterns with Envoy configuration.

Canary deployment with traffic splitting:

```yaml
routes:
  - match:
      prefix: "/reviews"
    route:
      weighted_clusters:
        clusters:
          - name: reviews_v1
            weight: 80
          - name: reviews_v2
            weight: 15
          - name: reviews_v3
            weight: 5
```

Header-based routing for A/B testing:

```yaml
routes:
  - match:
      prefix: "/reviews"
      headers:
        - name: "x-user-group"
          exact_match: "beta"
    route:
      cluster: reviews_v3
  - match:
      prefix: "/reviews"
    route:
      cluster: reviews_v1
```

Fault injection for resilience testing:

```yaml
routes:
  - match:
      prefix: "/reviews"
    route:
      cluster: reviews_v1
    typed_per_filter_config:
      envoy.filters.http.fault:
        "@type": type.googleapis.com/envoy.extensions.filters.http.fault.v3.HTTPFault
        delay:
          fixed_delay: 5s
          percentage:
            numerator: 10
            denominator: HUNDRED
        abort:
          http_status: 503
          percentage:
            numerator: 5
            denominator: HUNDRED
```

## Observability Stack

Deploy a complete observability stack alongside the mesh:

```yaml
# observability.yml
version: "3"
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "4317:4317"
```

Configure Envoy to export telemetry:

```yaml
# Add tracing configuration
tracing:
  http:
    name: envoy.tracers.opentelemetry
    typed_config:
      "@type": type.googleapis.com/envoy.config.trace.v3.OpenTelemetryConfig
      grpc_service:
        envoy_grpc:
          cluster_name: otel_collector
        timeout: 5s
      service_name: productpage

clusters:
  - name: otel_collector
    type: STRICT_DNS
    connect_timeout: 5s
    load_assignment:
      cluster_name: otel_collector
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: jaeger
                    port_value: 4317
```

## Mesh Management Script

Create a script to manage the service mesh:

```bash
#!/bin/bash
# mesh-ctl.sh

ACTION="${1:-status}"

case "$ACTION" in
    deploy)
        echo "Deploying service mesh..."
        podman-compose -f bookinfo-mesh.yml up -d
        podman-compose -f observability.yml up -d
        echo "Mesh deployed. Services available at http://localhost:9080"
        echo "Grafana: http://localhost:3000"
        echo "Jaeger: http://localhost:16686"
        ;;
    status)
        echo "=== Service Mesh Status ==="
        echo ""
        echo "Pods:"
        podman pod ps
        echo ""
        echo "Containers:"
        podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
        echo ""
        echo "Envoy Admin:"
        for proxy in productpage-proxy reviews-proxy details-proxy ratings-proxy; do
            PORT=$(podman port "$proxy" 15000 2>/dev/null | cut -d: -f2)
            if [ -n "$PORT" ]; then
                CLUSTERS=$(curl -s "http://localhost:$PORT/clusters" | grep -c "::healthy")
                echo "  $proxy: $CLUSTERS healthy upstream(s)"
            fi
        done
        ;;
    canary)
        VERSION="$2"
        WEIGHT="${3:-10}"
        echo "Setting canary for reviews to v${VERSION} at ${WEIGHT}%..."
        # Update Envoy configuration for traffic splitting
        ;;
    destroy)
        echo "Destroying service mesh..."
        podman-compose -f bookinfo-mesh.yml down
        podman-compose -f observability.yml down
        echo "Mesh destroyed"
        ;;
    *)
        echo "Usage: $0 {deploy|status|canary|destroy}"
        exit 1
        ;;
esac
```

## Rate Limiting Across Services

Implement mesh-wide rate limiting:

```yaml
http_filters:
  - name: envoy.filters.http.local_ratelimit
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
      stat_prefix: mesh_rate_limit
      token_bucket:
        max_tokens: 1000
        tokens_per_fill: 1000
        fill_interval: 60s
  - name: envoy.filters.http.router
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

## Health Checking and Circuit Breaking

Configure health checking across the mesh:

```yaml
clusters:
  - name: reviews_service
    connect_timeout: 5s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    health_checks:
      - timeout: 2s
        interval: 10s
        healthy_threshold: 2
        unhealthy_threshold: 3
        http_health_check:
          path: /health
    circuit_breakers:
      thresholds:
        - max_connections: 100
          max_pending_requests: 50
          max_requests: 200
          max_retries: 3
    outlier_detection:
      consecutive_5xx: 3
      interval: 10s
      base_ejection_time: 30s
      max_ejection_percent: 30
    load_assignment:
      cluster_name: reviews_service
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: reviews-proxy
                    port_value: 9080
```

## Conclusion

While Istio is designed for Kubernetes, its core patterns of sidecar proxies, traffic management, mTLS security, and observability can be implemented with Podman using Envoy proxies. Podman pods provide the shared network namespace needed for the sidecar pattern, and Envoy configuration gives you the same traffic management capabilities that Istio provides. This approach is valuable for environments where Kubernetes is not available or appropriate, but you still need the service mesh capabilities that modern microservice architectures demand. By combining Podman's pod model with Envoy's proxy capabilities and Istio's architectural patterns, you get a practical service mesh solution for container-based microservices.
