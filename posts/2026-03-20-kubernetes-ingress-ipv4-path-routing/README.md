# How to Set Up Kubernetes Ingress for IPv4 Path-Based Routing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, Ingress, IPv4, Path Routing, Nginx, Microservices

Description: Configure Kubernetes Ingress to route IPv4 HTTP requests to different backend services based on URL path, enabling microservice routing under a single hostname.

Path-based routing allows a single hostname to serve multiple microservices at different URL paths. This is common for API gateways and microservice architectures.

## Basic Path-Based Ingress

```yaml
# path-based-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      # /users/* routes to user-service
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      # /products/* routes to product-service
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 80
      # /orders/* routes to order-service
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      # / (root) routes to frontend
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 3000
```

## Path Rewriting

Without rewriting, `/users/123` is sent to the backend as `/users/123`. With rewrite, you can strip the prefix:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    # Capture group from path used in rewrite-target
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      # /api/v1/users/123 → sent to backend as /users/123
      - path: /api/v1(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

## Exact vs. Prefix Path Types

```yaml
spec:
  rules:
  - host: example.com
    http:
      paths:
      # Exact: only matches /login exactly (not /login/callback)
      - path: /login
        pathType: Exact
        backend:
          service:
            name: auth-service
            port:
              number: 80
      # Prefix: matches /api and /api/anything
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

## Testing Path Routing

```bash
INGRESS_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test each path
curl -H "Host: api.example.com" http://$INGRESS_IP/users
curl -H "Host: api.example.com" http://$INGRESS_IP/products
curl -H "Host: api.example.com" http://$INGRESS_IP/orders
curl -H "Host: api.example.com" http://$INGRESS_IP/

# Verify routing in Nginx config
kubectl exec -n ingress-nginx \
  $(kubectl get pods -n ingress-nginx -o name | head -1) \
  -- nginx -T | grep -A5 "location /users"
```

## Nginx Ingress Rate Limiting per Path

```yaml
metadata:
  annotations:
    # Limit to 100 requests per minute for /api
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "20"
```

## Priority of Rules

The Nginx Ingress Controller processes rules in this order:
1. Exact path matches
2. Longer prefix matches take priority over shorter ones
3. For equal-length paths, the first rule wins

Path-based routing reduces infrastructure complexity by consolidating multiple services under a single external IPv4 address and hostname.
