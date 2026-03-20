# How to Configure mTLS Between Services in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, mTLS, Security, Service Mesh, TLS

Description: Configure mutual TLS (mTLS) between services in Rancher-managed clusters to ensure all inter-service communication is authenticated and encrypted.

## Introduction

Mutual TLS (mTLS) ensures that both the client and server authenticate each other using certificates, providing strong identity verification and encryption for all service-to-service communication. In Kubernetes, mTLS is typically implemented via a service mesh or through manual certificate management. This guide covers both approaches in Rancher-managed environments.

## Prerequisites

- Rancher-managed cluster with Istio or Linkerd installed
- cert-manager for certificate management
- kubectl with cluster-admin access
- Basic understanding of TLS/PKI

## Understanding mTLS in Kubernetes

Standard TLS: Server proves its identity to the client.
mTLS: Both server AND client prove their identities to each other using certificates.

## Method 1: mTLS with Istio (Recommended)

### Enable Strict mTLS for the Entire Mesh

```yaml
# peer-authentication-global.yaml - Enforce mTLS mesh-wide
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  # Applying to istio-system affects the entire mesh
  namespace: istio-system
spec:
  mtls:
    # STRICT: reject all non-mTLS traffic
    mode: STRICT
```

### Enable mTLS Per Namespace

```yaml
# peer-authentication-ns.yaml - Namespace-scoped mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: namespace-mtls
  namespace: production
spec:
  mtls:
    mode: STRICT
```

### Enable mTLS Per Workload

```yaml
# peer-authentication-workload.yaml - Workload-specific mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: backend-mtls
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  mtls:
    mode: STRICT
  # Override mTLS mode for a specific port (e.g., for health checks)
  portLevelMtls:
    8080:
      mode: STRICT
    9090:  # Metrics port - allow plaintext
      mode: PERMISSIVE
```

### Verify Istio mTLS is Working

```bash
# Check the PeerAuthentication policy
kubectl get peerauthentication -n production

# Check that all pods have Istio sidecar injected
kubectl get pods -n production -o jsonpath='{range .items[*]}{.metadata.name}: {range .spec.containers[*]}{.name} {end}{"\n"}{end}'

# Verify mTLS using istioctl
istioctl authn tls-check <pod-name>.production

# View certificate details
istioctl proxy-config secret <pod-name>.production
```

## Method 2: mTLS with Linkerd

Linkerd enables mTLS by default for all meshed services:

```bash
# Verify mTLS is active for a service
linkerd viz edges deployment/backend -n production

# Check certificate details
linkerd diagnostics proxy-metrics -n production pod/<pod-name>

# View TLS connection details
kubectl exec -n production deployment/backend -- \
  curl -s http://localhost:4191/metrics | grep "inbound_http_route"
```

```yaml
# linkerd-mtls-annotation.yaml - Ensure mTLS is enabled for a specific pod
apiVersion: v1
kind: Pod
metadata:
  name: backend
  annotations:
    # Explicitly require mTLS
    config.linkerd.io/proxy-inject: enabled
```

## Method 3: Manual mTLS with cert-manager

For services not using a service mesh:

```yaml
# cert-manager-ca-issuer.yaml - Create an internal CA
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    # Reference the CA certificate secret
    secretName: internal-ca-secret
---
# service-certificate.yaml - Certificate for backend service
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: backend-tls
  namespace: production
spec:
  # The secret to store the certificate
  secretName: backend-tls-secret
  duration: 24h
  # Auto-renew before expiry
  renewBefore: 1h
  subject:
    organizations:
      - example.com
  commonName: backend.production.svc.cluster.local
  dnsNames:
    - backend.production.svc.cluster.local
    - backend.production.svc
    - backend
  issuerRef:
    name: internal-ca
    kind: ClusterIssuer
```

Configure the application to use certificates:

```yaml
# backend-mtls-deployment.yaml - App with manual mTLS
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: backend
          image: registry.example.com/backend:v1.0
          ports:
            - containerPort: 8443  # HTTPS port
          env:
            # Configure TLS from mounted secrets
            - name: TLS_CERT_FILE
              value: /certs/tls.crt
            - name: TLS_KEY_FILE
              value: /certs/tls.key
            - name: TLS_CA_FILE
              value: /certs/ca.crt
            # Enable client certificate verification
            - name: MTLS_ENABLED
              value: "true"
          volumeMounts:
            - name: tls-certs
              mountPath: /certs
              readOnly: true
      volumes:
        - name: tls-certs
          secret:
            secretName: backend-tls-secret
```

## Step 4: Testing mTLS Connectivity

```bash
# Test with curl - mTLS should succeed
kubectl exec -n production deployment/frontend -- \
  curl -v \
  --cert /certs/tls.crt \
  --key /certs/tls.key \
  --cacert /certs/ca.crt \
  https://backend.production.svc.cluster.local:8443/health

# Test should FAIL without client cert (mTLS enforced)
kubectl exec -n production deployment/frontend -- \
  curl -v \
  https://backend.production.svc.cluster.local:8443/health
# Expected: SSL handshake failure
```

## Step 5: Authorization Policies with mTLS Identity

Use mTLS identity for authorization:

```yaml
# authorization-policy.yaml - AuthorizationPolicy using mTLS identity
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
    - from:
        - source:
            # Only allow requests with this specific identity
            principals:
              - "cluster.local/ns/production/sa/frontend-sa"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
```

## Step 6: Monitor mTLS Certificate Health

```bash
# Check certificate expiry across the cluster (Istio)
istioctl proxy-config secret --all-namespaces | \
  grep -v "ACTIVE" | head -20

# Check cert-manager certificate status
kubectl get certificates --all-namespaces

# Alert on expiring certificates
kubectl get certificates --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: {.status.conditions[?(@.type=="Ready")].message}{"\n"}{end}'
```

## Conclusion

mTLS is a foundational security control for microservice architectures. Service meshes like Istio and Linkerd make it trivial to enable mTLS for all inter-service communication with zero application code changes. For services outside the mesh, cert-manager provides automated certificate lifecycle management. Always combine mTLS with proper authorization policies to get the full benefit of strong service identity—authentication alone is not sufficient without access control.
