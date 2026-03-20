# How to Configure HAProxy Ingress Controller for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, HAProxy, Kubernetes, Ingress, Load Balancer, Dual-Stack

Description: Configure HAProxy Ingress Controller in Kubernetes to accept IPv6 connections, configure dual-stack load balancer services, and handle IPv6 client IP forwarding with trusted proxy configuration.

## Introduction

HAProxy Ingress Controller is a Kubernetes ingress controller built on HAProxy, known for its high performance and advanced load balancing features. IPv6 configuration for HAProxy Ingress involves enabling dual-stack service exposure, configuring HAProxy frontends to listen on IPv6, and setting trusted proxy CIDRs for correct client IP handling when behind IPv6 load balancers.

## Install HAProxy Ingress with IPv6 (Helm)

```yaml
# haproxy-ingress-values.yaml

controller:
  # Service configuration for dual-stack exposure
  service:
    type: LoadBalancer
    # Dual-stack service
    ipFamilyPolicy: PreferDualStack
    ipFamilies:
      - IPv4
      - IPv6
    # Cloud annotations for dual-stack LB
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-ip-address-type: "dualstack"

  # HAProxy configuration
  config:
    # HAProxy frontend binds to all interfaces (IPv4 and IPv6)
    # when the pod has IPv6 connectivity
    bind-ipv4-address: "0.0.0.0"
    bind-ipv6-address: "::"

    # Trusted proxy CIDRs (IPv4 and IPv6)
    forwardfor: "if-missing"
    proxy-real-ip-header: "X-Forwarded-For"
    proxy-protocol: ""

    # Source IP extraction when behind IPv6 LB
    # Trusted IPs: allow these to set X-Forwarded-For
    # (HAProxy handles this via frontend TCP accept rules)
```

```bash
# Install HAProxy Ingress Controller
helm repo add haproxy-ingress https://haproxy-ingress.github.io/charts
helm install haproxy-ingress haproxy-ingress/haproxy-ingress \
    -n ingress-controller \
    --create-namespace \
    -f haproxy-ingress-values.yaml

# Verify service has IPv6 external IP
kubectl get svc haproxy-ingress -n ingress-controller
```

## Kubernetes Ingress with HAProxy

```yaml
# ingress-haproxy-ipv6.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "haproxy"
    # HAProxy-specific annotations
    haproxy-ingress.github.io/balance-algorithm: "leastconn"
    haproxy-ingress.github.io/timeout-connect: "5s"
    haproxy-ingress.github.io/timeout-server: "60s"
    # IPv6 forwarding configuration
    haproxy-ingress.github.io/forwardfor: "add"
    haproxy-ingress.github.io/proxy-real-ip-header: "X-Real-IP"
spec:
  ingressClassName: haproxy
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 8080
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
```

## HAProxy ConfigMap for IPv6

```yaml
# haproxy-ingress-configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: haproxy-ingress
  namespace: ingress-controller
data:
  # Global HAProxy settings for IPv6

  # Bind to both IPv4 and IPv6
  bind-ipv4-address: "0.0.0.0"
  bind-ipv6-address: "::"

  # Client IP forwarding
  forwardfor: "add"
  use-forward-for: "true"

  # Trusted proxies: IPs allowed to set X-Forwarded-For
  # Specified as whitespace-separated CIDRs
  whitelist-source-range: "10.0.0.0/8 fd00::/8 2001:db8:lb::/48"

  # HTTPS redirect
  ssl-redirect: "true"
  http-port: "80"
  https-port: "443"

  # Real client IP extraction
  real-ip-header: "X-Forwarded-For"
```

## HAProxy Backend Configuration for IPv6

```yaml
# haproxy-backend-ipv6.yaml — Configure specific backend for IPv6

apiVersion: haproxy-ingress.github.io/v1
kind: Backend
metadata:
  name: myapp-backend
  namespace: production
spec:
  # Backend connection settings
  timeoutConnect: "5s"
  timeoutServer: "60s"

  # Health check
  healthCheck:
    enabled: true
    uri: "/health"
    interval: "10s"

  # Source IP configuration for IPv6 backends
  # HAProxy connects to backend pods via their pod IPs
  # In dual-stack clusters, pods have both IPv4 and IPv6 addresses
```

## PROXY Protocol v2 Configuration (HAProxy to Backend)

```yaml
# haproxy-proxy-protocol.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-proxy-proto
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "haproxy"
    # Send PROXY Protocol v2 headers to backends
    # Enables backends to see real IPv6 client addresses
    haproxy-ingress.github.io/proxy-protocol: "v2"
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-proxy-aware
                port:
                  number: 8080
```

## Rate Limiting IPv6 Clients in HAProxy Ingress

```yaml
# haproxy-rate-limit.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-rate-limited
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "haproxy"
    # Rate limiting (applies to all clients including IPv6)
    haproxy-ingress.github.io/limit-rps: "100"
    haproxy-ingress.github.io/limit-whitelist: "10.0.0.0/8,fd00::/8"
    # Scale the limit window
    haproxy-ingress.github.io/limit-period: "60"
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 8080
```

## Verify HAProxy Ingress IPv6

```bash
# Check HAProxy pod listens on IPv6
kubectl exec -n ingress-controller deployment/haproxy-ingress -- \
    ss -tlnp | grep -E ":80|:443"
# Should show [::]:80 and [::]:443

# Test HTTP access over IPv6
curl -6 -H "Host: app.example.com" \
    "http://[2001:db8::haproxy-lb]:80/"

# Check HAProxy stats page
kubectl port-forward -n ingress-controller svc/haproxy-ingress 1936:1936
curl http://localhost:1936/stats

# Check real client IP is passed (connect from IPv6 client)
curl -6 -H "Host: app.example.com" \
    "http://[2001:db8::haproxy-lb]:80/api/ip"
# Expected: {"ip": "2001:db8::client", "version": 6}

# Check HAProxy configuration rendered for the ingress
kubectl exec -n ingress-controller deployment/haproxy-ingress -- \
    cat /etc/haproxy/haproxy.cfg | grep -A5 "frontend\|bind"
```

## Conclusion

HAProxy Ingress Controller exposes IPv6 by binding frontends to `[::]` using `bind-ipv6-address: "::"` in the ConfigMap and setting the Kubernetes service to `ipFamilyPolicy: PreferDualStack`. The `forwardfor` option adds `X-Forwarded-For` headers with the real client IPv6 address. PROXY Protocol v2 can carry IPv6 client addresses to backends that support it, using the `haproxy-ingress.github.io/proxy-protocol: "v2"` annotation. Rate limiting applies to IPv6 clients via the `limit-rps` annotation with IPv6 CIDR whitelisting. Trusted proxy configuration uses the `whitelist-source-range` ConfigMap option with both IPv4 and IPv6 CIDR ranges for correct client IP extraction when HAProxy sits behind a cloud load balancer.
