# How to Use Dapr with Envoy as API Gateway

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Envoy, Gateway, Proxy, Kubernetes

Description: Learn how to configure Envoy proxy as an API gateway for Dapr-enabled microservices, with route matching, header manipulation, and cluster configuration.

---

## Overview

Envoy is a high-performance edge and service proxy used as the data plane for many service meshes and API gateways. Using Envoy in front of Dapr-enabled services gives you fine-grained control over routing, load balancing, and observability at the request level.

## Envoy Configuration Structure

Envoy is configured through a bootstrap YAML that defines listeners, route configurations, and clusters. A typical setup routes external HTTP to Dapr sidecar ports:

```yaml
static_resources:
  listeners:
    - name: api_listener
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 8080
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: dapr_services
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/orders"
                          route:
                            cluster: order_service_dapr
                            prefix_rewrite: "/v1.0/invoke/order-service/method"
                        - match:
                            prefix: "/users"
                          route:
                            cluster: user_service_dapr
                            prefix_rewrite: "/v1.0/invoke/user-service/method"
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

## Defining Upstream Clusters

Each Dapr sidecar endpoint is a cluster in Envoy:

```yaml
  clusters:
    - name: order_service_dapr
      connect_timeout: 1s
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: order_service_dapr
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: order-service.default.svc.cluster.local
                      port_value: 3500
    - name: user_service_dapr
      connect_timeout: 1s
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: user_service_dapr
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: user-service.default.svc.cluster.local
                      port_value: 3500
```

## Adding Request Headers for Dapr

Inject the `dapr-app-id` header to help with routing and tracing:

```yaml
routes:
  - match:
      prefix: "/orders"
    route:
      cluster: order_service_dapr
    request_headers_to_add:
      - header:
          key: dapr-app-id
          value: order-service
        keep_empty_value: false
```

## Deploying Envoy as a Kubernetes ConfigMap

Store the Envoy config in a ConfigMap and mount it to the Envoy pod:

```bash
kubectl create configmap envoy-config \
  --from-file=envoy.yaml=./envoy-config.yaml \
  -n default
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: envoy-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: envoy-gateway
  template:
    metadata:
      labels:
        app: envoy-gateway
    spec:
      containers:
        - name: envoy
          image: envoyproxy/envoy:v1.28-latest
          args:
            - "-c"
            - "/etc/envoy/envoy.yaml"
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy
      volumes:
        - name: envoy-config
          configMap:
            name: envoy-config
```

## Enabling Access Logs

Add access logging to Envoy for request-level observability:

```yaml
access_log:
  - name: envoy.access_loggers.stdout
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
      log_format:
        json_format:
          start_time: "%START_TIME%"
          method: "%REQ(:METHOD)%"
          path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
          response_code: "%RESPONSE_CODE%"
          duration: "%DURATION%"
```

## Testing the Setup

Verify routing through Envoy to Dapr services:

```bash
# Route to order service via Envoy
curl http://envoy-gateway.default.svc:8080/orders/list

# Route to user service
curl http://envoy-gateway.default.svc:8080/users/profile
```

## Summary

Envoy provides a powerful, low-level API gateway option for Dapr-enabled microservices. Its flexible routing rules, header manipulation, and cluster definitions give you precise control over how external requests reach Dapr sidecars. Combined with Dapr's service invocation model, Envoy delivers high-performance edge routing with deep observability.
