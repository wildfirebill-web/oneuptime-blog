# How to Set Up a Service Mesh in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Service Mesh

Description: Learn how to install and configure a service mesh in Rancher using Istio for advanced traffic management, observability, and security.

A service mesh provides a dedicated infrastructure layer for handling service-to-service communication. It adds capabilities like traffic management, observability, mutual TLS, and circuit breaking without requiring changes to your application code. Rancher has built-in support for Istio, making it straightforward to deploy a service mesh. This guide walks through the setup.

## Prerequisites

- A running Rancher instance (v2.6 or later)
- A managed Kubernetes cluster with at least 3 nodes
- At least 4 CPU cores and 8 GB RAM available for Istio components
- kubectl access to your cluster

## Step 1: Install Istio via Rancher

Rancher provides Istio as a built-in chart:

1. Navigate to your cluster in the Rancher dashboard.
2. Go to **Apps** > **Charts**.
3. Search for **Istio**.
4. Click **Install**.
5. Configure installation options:
   - Enable or disable Kiali (visualization dashboard)
   - Enable or disable Jaeger (distributed tracing)
   - Configure resource limits
6. Click **Install**.

## Step 2: Install Istio via Helm

Alternatively, install using istioctl or Helm:

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

# Install Istio base (CRDs)
helm install istio-base istio/base \
  --namespace istio-system \
  --create-namespace

# Install istiod (control plane)
helm install istiod istio/istiod \
  --namespace istio-system

# Install ingress gateway
helm install istio-ingress istio/gateway \
  --namespace istio-system
```

Verify the installation:

```bash
kubectl get pods -n istio-system
```

## Step 3: Enable Sidecar Injection

Label namespaces to enable automatic sidecar injection:

```bash
kubectl label namespace default istio-injection=enabled
```

Verify the label:

```bash
kubectl get namespace default --show-labels
```

All new pods in the labeled namespace will automatically get an Envoy sidecar proxy.

## Step 4: Deploy a Sample Application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        version: v1
    spec:
      containers:
      - name: frontend
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: default
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

After deploying, verify the sidecar was injected:

```bash
kubectl get pods -n default
```

Each pod should show 2/2 containers (your app plus the Envoy sidecar).

## Step 5: Configure an Istio Gateway

Create a gateway for external traffic:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: app-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "myapp.example.com"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: frontend-vs
  namespace: default
spec:
  hosts:
  - "myapp.example.com"
  gateways:
  - app-gateway
  http:
  - route:
    - destination:
        host: frontend
        port:
          number: 80
```

## Step 6: Configure Traffic Splitting

Split traffic between service versions for canary deployments:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: frontend-vs
  namespace: default
spec:
  hosts:
  - frontend
  http:
  - route:
    - destination:
        host: frontend
        subset: v1
      weight: 90
    - destination:
        host: frontend
        subset: v2
      weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: frontend-dr
  namespace: default
spec:
  host: frontend
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

## Step 7: Enable Mutual TLS

Enforce mTLS for all service-to-service communication:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
```

For cluster-wide enforcement:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

## Step 8: Configure Circuit Breaking

Protect services from cascading failures:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend-circuit-breaker
  namespace: default
spec:
  host: backend-api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

## Step 9: Access Kiali Dashboard

If Kiali was installed, access the service mesh visualization:

```bash
kubectl port-forward svc/kiali -n istio-system 20001:20001
```

Open `http://localhost:20001` in your browser to view:

- Service topology graphs
- Traffic flow between services
- Error rates and latency
- Configuration validation

## Step 10: Configure Retries and Timeouts

Add resilience to your services:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend-vs
  namespace: default
spec:
  hosts:
  - backend-api
  http:
  - route:
    - destination:
        host: backend-api
    timeout: 10s
    retries:
      attempts: 3
      perTryTimeout: 3s
      retryOn: 5xx,reset,connect-failure
```

## Troubleshooting

- **Sidecar not injected**: Verify the namespace label: `kubectl get ns --show-labels`
- **503 errors**: Check DestinationRule and VirtualService configurations
- **mTLS issues**: Verify PeerAuthentication policies: `kubectl get peerauthentication --all-namespaces`
- **Gateway not working**: Check the Istio ingress gateway pod: `kubectl get pods -n istio-system`
- **View Envoy config**: `kubectl exec <pod> -c istio-proxy -- pilot-agent request GET config_dump`

## Summary

Setting up a service mesh with Istio in Rancher provides advanced traffic management, security through mutual TLS, and deep observability into your microservices architecture. Rancher simplifies the installation and management of Istio, while Kiali provides a visual dashboard for monitoring service mesh traffic. Start with basic traffic routing and gradually adopt features like circuit breaking, retries, and traffic splitting as your needs grow.
