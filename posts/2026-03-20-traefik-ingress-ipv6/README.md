# How to Configure Traefik Ingress Controller for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Traefik, Kubernetes, Ingress, Dual-Stack, IngressRoute

Description: Configure Traefik Ingress Controller in Kubernetes to accept IPv6 traffic, expose services over IPv6 load balancers, and handle IPv6 client IP forwarding in Kubernetes ingress configurations.

## Introduction

Traefik is a cloud-native ingress controller for Kubernetes that automatically discovers services and configures routing. For IPv6, Traefik must be configured to listen on IPv6 entry points, and the Kubernetes service exposing Traefik must have an IPv6 external IP. Both standard Ingress resources and Traefik's native IngressRoute CRDs support IPv6 backends.

## Install Traefik with IPv6 Entry Points (Helm)

```yaml
# traefik-values.yaml - Helm values for Traefik with IPv6

deployment:
  replicas: 2

service:
  # Dual-stack service for Traefik's LoadBalancer
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv4
    - IPv6

ports:
  web:
    port: 80
  websecure:
    port: 443
    tls:
      enabled: true

# Traefik listens on all interfaces (IPv4 and IPv6) by default

# with [::]:port when the container has IPv6
additionalArguments:
  - "--entrypoints.web.address=:80"
  - "--entrypoints.websecure.address=:443"
  - "--entrypoints.web.forwardedHeaders.trustedIPs=10.0.0.0/8,fd00::/8,2001:db8::/32"
  - "--entrypoints.websecure.forwardedHeaders.trustedIPs=10.0.0.0/8,fd00::/8,2001:db8::/32"
```

```bash
# Install Traefik with IPv6 Helm values
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik \
    -n traefik-system \
    --create-namespace \
    -f traefik-values.yaml

# Verify Traefik service has IPv6 external IP
kubectl get svc traefik -n traefik-system -o wide
# Should show IPv6 in EXTERNAL-IP column (from cloud load balancer)
```

## Standard Ingress for IPv6

```yaml
# ingress-ipv6.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    # Strip the X-Forwarded-For header and use the real client IP
    traefik.ingress.kubernetes.io/router.middlewares: production-real-ip@kubernetescrd
spec:
  ingressClassName: traefik
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
      secretName: app-tls-cert
```

## Traefik IngressRoute for IPv6

```yaml
# ingressroute-ipv6.yaml

apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: myapp
  namespace: production
spec:
  entryPoints:
    - websecure

  routes:
    - match: Host(`app.example.com`)
      kind: Rule
      services:
        - name: myapp
          port: 8080

      # Middleware for IPv6 client IP handling
      middlewares:
        - name: real-ip-ipv6

  tls:
    certResolver: letsencrypt

---
# Middleware to handle IPv6 X-Forwarded-For
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: real-ip-ipv6
  namespace: production
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: "https"
```

## Traefik with IPv6 IP Allowlist

```yaml
# ipv6-allowlist-middleware.yaml

apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: ipv6-allowlist
  namespace: production
spec:
  ipAllowList:
    sourceRange:
      - "fd00::/8"            # Internal ULA
      - "2001:db8:corp::/48"  # Corporate IPv6
      - "10.0.0.0/8"          # IPv4 internal (dual-stack)
    ipStrategy:
      depth: 1   # Skip 1 proxy (the load balancer)
```

```yaml
# Apply to IngressRoute
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: internal-api
  namespace: production
spec:
  routes:
    - match: Host(`internal.example.com`) && PathPrefix(`/api`)
      kind: Rule
      middlewares:
        - name: ipv6-allowlist
      services:
        - name: internal-api
          port: 8080
```

## Traefik Service for Dual-Stack

```yaml
# traefik-service-dualstack.yaml

# Ensure backend services are dual-stack
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv4
    - IPv6
  selector:
    app: myapp
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

## Verify Traefik IPv6 Operation

```bash
# Check Traefik pod has IPv6
kubectl exec -n traefik-system deployment/traefik -- \
    ip -6 addr show

# Check Traefik entry points are listening on IPv6
kubectl exec -n traefik-system deployment/traefik -- \
    ss -tlnp | grep ":80\|:443"
# Should show [::]:80 and [::]:443

# Test HTTP access over IPv6
curl -6 -H "Host: app.example.com" "http://[2001:db8::traefik]:80/"

# Check that Traefik passes correct client IPv6
curl -6 -H "Host: app.example.com" \
     "http://[2001:db8::traefik]:80/api/ip"
# Expected: {"ip": "2001:db8::client", "version": 6}

# View Traefik dashboard (port-forward)
kubectl port-forward -n traefik-system svc/traefik 9000:9000 &
curl http://localhost:9000/dashboard/
```

## Conclusion

Traefik Ingress Controller supports IPv6 by listening on all interfaces via `[:]:port` in entry point configuration. The Kubernetes service exposing Traefik uses `ipFamilyPolicy: PreferDualStack` to obtain both IPv4 and IPv6 external addresses from the cloud load balancer. Standard Kubernetes Ingress resources and Traefik IngressRoute CRDs both route to IPv6-capable backend services without IPv6-specific configuration. The `forwardedHeaders.trustedIPs` configuration must include both IPv4 and IPv6 CIDR ranges for correct client IP extraction from X-Forwarded-For. IP allowlist middleware supports IPv6 CIDRs natively via the `sourceRange` field.
