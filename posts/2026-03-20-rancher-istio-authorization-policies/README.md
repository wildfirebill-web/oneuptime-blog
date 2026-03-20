# How to Configure Istio Authorization Policies in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Istio, Authorization, Security, Service Mesh

Description: Learn how to configure Istio AuthorizationPolicies to implement fine-grained access control for services in Rancher-managed Kubernetes clusters.

Istio AuthorizationPolicies provide access control for services in the mesh, implementing a zero-trust security model where access is denied by default and must be explicitly allowed. Unlike Kubernetes RBAC, which controls access to Kubernetes API resources, Istio AuthorizationPolicies control access to services within the mesh based on service identity, HTTP attributes, and other conditions. This guide covers how to implement authorization policies in a Rancher environment.

## Prerequisites

- Istio installed with mTLS enabled in your Rancher cluster
- Applications deployed with sidecar injection
- Basic understanding of Istio service identities (SPIFFE/X.509)
- `kubectl` access to the cluster

## Understanding Authorization Policy Actions

Istio AuthorizationPolicies support three actions:

- **ALLOW**: Explicitly allow matching requests
- **DENY**: Explicitly deny matching requests
- **CUSTOM**: Delegate to an external authorization system

## Step 1: Enable Deny-All Policy (Zero Trust)

Start with a default deny-all policy and explicitly allow required traffic:

```yaml
# deny-all.yaml - Block all traffic in a namespace by default

apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: my-app
spec:
  # Empty spec with no rules means deny all traffic
  {}
```

```bash
kubectl apply -f deny-all.yaml

# Verify traffic is now blocked
# Try to access a service and confirm you get a 403 Forbidden response
```

## Step 2: Allow Traffic from Specific Services

```yaml
# allow-frontend-to-backend.yaml - Allow frontend to call backend
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: my-app
spec:
  selector:
    matchLabels:
      # Apply this policy to the backend service
      app: backend
  action: ALLOW
  rules:
  - from:
    # Allow requests from the frontend service account
    - source:
        principals:
        - "cluster.local/ns/my-app/sa/frontend-service-account"
    to:
    - operation:
        # Only allow GET and POST methods
        methods: ["GET", "POST"]
        # Only allow access to /api/* paths
        paths: ["/api/*"]
```

## Step 3: Namespace-Level Access Control

```yaml
# allow-namespace.yaml - Allow all traffic from a specific namespace
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-from-frontend-namespace
  namespace: backend-ns
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        # Allow any service from the frontend namespace
        namespaces: ["frontend-ns"]
```

## Step 4: HTTP Attribute-Based Access Control

```yaml
# http-attribute-policy.yaml - Control access based on HTTP attributes
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: http-headers-policy
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: my-api
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/my-app/sa/trusted-service"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/public/*"]
    when:
    # Additional condition: require a specific header
    - key: request.headers[x-api-version]
      values: ["v2"]
```

## Step 5: JWT-Based Authorization

Control access using JWT token claims:

```yaml
# jwt-policy.yaml - Require valid JWT and check claims
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: my-api
  jwtRules:
  - issuer: "https://auth.example.com"
    jwksUri: "https://auth.example.com/.well-known/jwks.json"
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: jwt-authorization
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: my-api
  action: ALLOW
  rules:
  - from:
    - source:
        # Require a valid JWT token
        requestPrincipals: ["*"]
    when:
    # Only allow users with the admin role
    - key: request.auth.claims[role]
      values: ["admin"]
```

## Step 6: Ingress Gateway Authorization

Control what external traffic can reach your services:

```yaml
# ingress-policy.yaml - Control traffic from the ingress gateway
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-ingress-gateway
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: my-frontend
  action: ALLOW
  rules:
  - from:
    - source:
        # Only allow traffic from the ingress gateway
        principals:
        - "cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
```

## Step 7: Deny Specific Operations

```yaml
# deny-admin-from-external.yaml - Deny admin endpoints from specific sources
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-external-admin-access
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: my-api
  action: DENY
  rules:
  - to:
    - operation:
        # Deny access to admin paths from remote IPs
        paths: ["/admin/*"]
    when:
    # Only deny when NOT coming from internal network
    - key: source.ip
      notValues: ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"]
```

## Step 8: Verify Authorization Policies

```bash
# List all authorization policies
kubectl get authorizationpolicy -A

# Describe a specific policy
kubectl describe authorizationpolicy allow-frontend-to-backend -n my-app

# Check Envoy proxy logs for authorization decisions
kubectl logs -n my-app -c istio-proxy \
  $(kubectl get pod -n my-app -l app=backend -o jsonpath='{.items[0].metadata.name}') \
  | grep -E "403|RBAC|authz"

# Enable debug logging for authorization
kubectl exec -n my-app -c istio-proxy \
  $(kubectl get pod -n my-app -l app=backend -o jsonpath='{.items[0].metadata.name}') \
  -- curl -X POST localhost:15000/logging?rbac=debug
```

## Conclusion

Istio AuthorizationPolicies implement a powerful zero-trust security model for your service mesh. By combining service identity (via mTLS), namespace-based controls, HTTP attribute matching, and JWT authentication, you can build a comprehensive access control system that protects your microservices at the network layer without requiring any changes to your application code. When deployed through Rancher, these policies can be consistently applied and monitored across multiple clusters.
