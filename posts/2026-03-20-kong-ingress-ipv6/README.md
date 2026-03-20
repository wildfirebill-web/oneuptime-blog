# How to Configure Kong Ingress Controller for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Kong, Kubernetes, Ingress, API Gateway, Dual-Stack

Description: Configure Kong Ingress Controller to accept IPv6 traffic, route to IPv6 backend services, and apply plugins for IPv6 client IP handling and rate limiting in dual-stack Kubernetes clusters.

## Introduction

Kong Ingress Controller (KIC) is a Kubernetes-native API gateway built on Kong Gateway. For IPv6, Kong must be configured to listen on IPv6 ports, and its Kubernetes service must expose IPv6 load balancer addresses. Kong plugins for rate limiting and IP restriction support IPv6 CIDR notation for proper IPv6 client IP handling.

## Install Kong with IPv6 Service (Helm)

```yaml
# kong-values.yaml — Helm values for Kong with IPv6

proxy:
  # Kong proxy service with dual-stack
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ip-address-type: "dualstack"

  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv4
    - IPv6

  # Proxy ports (Kong listens on all interfaces by default)
  http:
    enabled: true
    servicePort: 80
    containerPort: 8000

  tls:
    enabled: true
    servicePort: 443
    containerPort: 8443

admin:
  type: ClusterIP
  ipFamilyPolicy: SingleStack
  ipFamilies: [IPv6]   # Admin only on IPv6 for security
  http:
    enabled: true
    servicePort: 8001

# Kong environment for IPv6
env:
  # Kong admin listen on all interfaces (IPv6 and IPv4)
  admin_listen: "0.0.0.0:8001, [::]:8001"
  proxy_listen: "0.0.0.0:8000, [::]:8000, 0.0.0.0:8443 ssl, [::]:8443 ssl"
```

```bash
# Install Kong Ingress Controller
helm repo add kong https://charts.konghq.com
helm install kong kong/ingress -n kong --create-namespace -f kong-values.yaml

# Verify Kong service has IPv6
kubectl get svc -n kong kong-proxy -o jsonpath='{.status.loadBalancer.ingress}'
# Should show IPv6 address from cloud provider
```

## Standard Ingress with Kong

```yaml
# ingress-kong-ipv6.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: production
  annotations:
    # Kong-specific annotations
    konghq.com/strip-path: "false"
    konghq.com/protocols: "https"
    # Plugin for rate limiting (supports IPv6)
    konghq.com/plugins: rate-limit-ipv6,ip-restrict-ipv6
spec:
  ingressClassName: kong
  rules:
    - host: api.example.com
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
        - api.example.com
      secretName: api-tls
```

## Kong Plugin: Rate Limiting with IPv6

```yaml
# kong-rate-limit-ipv6.yaml

apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limit-ipv6
  namespace: production
plugin: rate-limiting
config:
  # Rate limit by /48 prefix for IPv6 (Kong uses IP by default)
  # For IPv6-aware rate limiting, use consumer or custom key
  minute: 100
  hour: 1000
  policy: local
  limit_by: ip   # Kong rate limits by IP (full address)
  # Note: for prefix-based IPv6 rate limiting, use lua-resty-limit-traffic plugin
```

## Kong Plugin: IP Restriction for IPv6

```yaml
# kong-ip-restrict-ipv6.yaml

apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ip-restrict-ipv6
  namespace: production
plugin: ip-restriction
config:
  # Allow list: both IPv4 and IPv6 CIDRs
  allow:
    - "10.0.0.0/8"
    - "192.168.0.0/16"
    - "fd00::/8"           # ULA internal
    - "2001:db8:corp::/48" # Corporate IPv6
  # Deny list (optional)
  deny:
    - "2001:db8:malicious::/48"
```

## KongIngress for IPv6 Backend Configuration

```yaml
# kong-ingress-ipv6.yaml

# Configure upstream for IPv6 backends
apiVersion: configuration.konghq.com/v1
kind: KongIngress
metadata:
  name: myapp-upstream-config
  namespace: production
upstream:
  # Load balancing algorithm
  algorithm: round-robin
  # Health check for IPv6 backends
  healthchecks:
    active:
      http_path: /health
      healthy:
        interval: 10
        successes: 2
      unhealthy:
        interval: 5
        http_failures: 3

# Route configuration
route:
  protocols:
    - https
  https_redirect_status_code: 308
  preserve_host: true

# Service configuration
service:
  protocol: http
  port: 8080
  retries: 5
  connect_timeout: 5000
  write_timeout: 60000
  read_timeout: 60000
```

## Kong Admin API Configuration via IPv6

```bash
# Access Kong Admin API over IPv6
curl -6 "http://[2001:db8::kong]:8001/services" | jq .

# Add a service with IPv6 backend via Admin API
curl -6 -X POST "http://[2001:db8::kong]:8001/services" \
    -H "Content-Type: application/json" \
    -d '{
        "name": "myapp",
        "url": "http://[2001:db8::app]:8080"
    }'

# Add route
curl -6 -X POST "http://[2001:db8::kong]:8001/services/myapp/routes" \
    -H "Content-Type: application/json" \
    -d '{
        "hosts": ["api.example.com"],
        "protocols": ["https"]
    }'

# Enable rate limiting plugin for IPv6
curl -6 -X POST "http://[2001:db8::kong]:8001/services/myapp/plugins" \
    -H "Content-Type: application/json" \
    -d '{
        "name": "rate-limiting",
        "config": {"minute": 100, "policy": "local"}
    }'
```

## Kong with IPv6 Real IP

```yaml
# kong-real-ip-plugin.yaml — Configure trusted IPs for X-Forwarded-For

apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: forwarded-ip
  namespace: production
plugin: request-transformer
config:
  # Ensure X-Real-IP is passed to backends
  add:
    headers:
      - "X-Real-IP:$(request.headers['x-forwarded-for'])"
```

```bash
# Set trusted CIDRs in Kong (for real IP determination)
# In kong.conf or via environment variable:
# trusted_ips = 10.0.0.0/8, fd00::/8, 2001:db8:lb::/48

# Helm values equivalent:
# env:
#   trusted_ips: "10.0.0.0/8,fd00::/8,2001:db8:lb::/48"
#   real_ip_header: "X-Forwarded-For"
```

## Verify Kong IPv6 Operation

```bash
# Check Kong proxy listens on IPv6
kubectl exec -n kong deployment/kong -- \
    ss -tlnp | grep -E ":8000|:8443"
# Should show [::]:8000 and [::]:8443

# Test Kong proxy over IPv6
curl -6 -H "Host: api.example.com" "http://[2001:db8::kong-lb]:80/health"

# Check Kong admin API
curl -6 "http://[2001:db8::kong]:8001/status" | jq .server.connections_active

# View Kong routes
kubectl exec -n kong deployment/kong -- \
    curl -s http://localhost:8001/routes | jq '.data[].hosts'
```

## Conclusion

Kong Ingress Controller supports IPv6 through `proxy_listen` and `admin_listen` configuration with `[::]:port` bindings. The Kubernetes service uses `ipFamilyPolicy: PreferDualStack` to obtain IPv6 load balancer addresses from the cloud provider. Standard Kubernetes Ingress resources and Kong's KongPlugin CRDs configure routing and plugins for IPv6 traffic. The `ip-restriction` plugin natively accepts IPv6 CIDR notation in allow/deny lists. Set `trusted_ips` in Kong's configuration to include IPv6 CIDR ranges for correct real IP extraction from `X-Forwarded-For` when Kong sits behind a load balancer.
