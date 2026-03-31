# How to Use Dapr with Kubernetes Gateway API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, Gateway API, Ingress, Networking, HTTPRoute

Description: Learn how to use the Kubernetes Gateway API with Dapr to route external traffic to Dapr-enabled services using HTTPRoute, GatewayClass, and traffic splitting.

---

## What Is the Kubernetes Gateway API?

The Kubernetes Gateway API is the successor to Ingress, providing richer traffic management capabilities. Key resources include:
- **GatewayClass** - Defines the controller managing the gateway
- **Gateway** - Configures the listener (port, protocol, TLS)
- **HTTPRoute** - Routes HTTP traffic to backend services

Dapr-enabled services can be targeted as backends in HTTPRoute rules.

## Installing a Gateway API Controller

Install NGINX Gateway Fabric as the Gateway API implementation:

```bash
# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

# Install NGINX Gateway Fabric
helm install ngf oci://ghcr.io/nginxinc/charts/nginx-gateway-fabric \
  --namespace nginx-gateway \
  --create-namespace
```

## Creating a Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: production-gateway
  namespace: production
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: production-tls-cert
```

## Routing to Dapr Services

Create an HTTPRoute that targets a Dapr-enabled service:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: order-service-route
  namespace: production
spec:
  parentRefs:
    - name: production-gateway
  hostnames:
    - "api.mycompany.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /orders
      backendRefs:
        - name: order-service
          port: 8080
          weight: 100
```

The Dapr sidecar on `order-service` intercepts the request and applies middleware, mTLS, and tracing automatically.

## Traffic Splitting with Gateway API

Use HTTPRoute weights for canary deployments of Dapr services:

```yaml
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /checkout
    backendRefs:
      - name: checkout-service-stable
        port: 8080
        weight: 90
      - name: checkout-service-canary
        port: 8080
        weight: 10
```

Both backends should be Dapr-enabled with the same state store and pub/sub components.

## Adding Request Modification Filters

Inject headers for Dapr service invocation context:

```yaml
rules:
  - matches:
      - path:
          type: Exact
          value: /api/inventory
    filters:
      - type: RequestHeaderModifier
        requestHeaderModifier:
          add:
            - name: "dapr-app-id"
              value: "inventory-service"
    backendRefs:
      - name: dapr-sidecar-proxy
        port: 3500
```

## Exposing Dapr Workflow API Externally

Route external workflow API calls through the Gateway:

```yaml
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /v1.0/workflow
    backendRefs:
      - name: workflow-orchestrator
        port: 3500    # Route directly to Dapr HTTP port
```

## Monitoring Gateway and Dapr Together

Combine Gateway API metrics with Dapr metrics in Grafana:

```bash
# NGINX Gateway Fabric metrics
nginx_gateway_fabric_nginx_requests_total

# Dapr service invocation metrics
dapr_service_invocation_req_sent_total

# Correlate: requests entering gateway vs processed by Dapr
```

## Summary

The Kubernetes Gateway API routes external traffic to Dapr-enabled services using HTTPRoute resources. Gateway-level features like traffic splitting for canary deployments, header injection, and TLS termination complement Dapr's application-level features. Together they provide a complete ingress-to-application platform where the Gateway handles external routing and Dapr handles internal service communication, state, and pub/sub.
