# How to Configure Linkerd with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linkerd, IPv6, Service Mesh, Dual-Stack, Kubernetes, Proxy

Description: A guide to configuring Linkerd service mesh with IPv6 and dual-stack Kubernetes clusters, including proxy injection, HTTPRoutes, and observability for IPv6 traffic.

Linkerd 2.12+ supports dual-stack IPv4/IPv6 Kubernetes clusters. The Linkerd proxy (written in Rust) handles IPv6 connections transparently once the cluster is configured for dual-stack.

## Prerequisites: Dual-Stack Kubernetes Cluster

Linkerd requires the underlying Kubernetes cluster to support dual-stack. Verify:

```bash
# Check the cluster has dual-stack service CIDRs
kubectl get configmap kubeadm-config -n kube-system -o yaml | grep -A 5 networking

# Check a pod has both IPv4 and IPv6 addresses
kubectl run check-ipv6 --image=busybox --restart=Never -- sleep 60
kubectl get pod check-ipv6 -o jsonpath='{.status.podIPs}'
# Output: [{"ip":"10.0.0.5"},{"ip":"fd00::5"}]
kubectl delete pod check-ipv6
```

## Installing Linkerd on a Dual-Stack Cluster

```bash
# Install Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin

# Verify pre-installation checks (checks dual-stack if present)
linkerd check --pre

# Install Linkerd CRDs
linkerd install --crds | kubectl apply -f -

# Install Linkerd control plane
linkerd install | kubectl apply -f -

# Verify installation
linkerd check
```

## Verifying IPv6 in Linkerd Control Plane

```bash
# Check Linkerd control plane pods have dual-stack IPs
kubectl get pods -n linkerd -o wide
kubectl get pod -n linkerd -l linkerd.io/control-plane-component=destination \
  -o jsonpath='{.items[0].status.podIPs}'

# Check Linkerd destination service has dual-stack ClusterIP
kubectl get svc -n linkerd linkerd-dst \
  -o jsonpath='{.spec.clusterIPs}'

# Inspect the Linkerd proxy config for IPv6
kubectl get cm -n linkerd linkerd-config -o yaml | grep -i ipv6
```

## Injecting Linkerd Proxy into Deployments

```yaml
# Deployment with Linkerd proxy injection
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
  annotations:
    linkerd.io/inject: enabled    # Inject Linkerd proxy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        linkerd.io/inject: enabled
    spec:
      containers:
        - name: app
          image: nginx:alpine
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml

# Verify proxy was injected (2/2 containers)
kubectl get pod -l app=my-app
# NAME                      READY   STATUS
# my-app-xxx-yyy            2/2     Running

# Check proxy has IPv6 listener
kubectl exec -c linkerd-proxy \
  $(kubectl get pod -l app=my-app -o name | head -1) \
  -- ss -6 -tlnp | grep 4143
```

## Linkerd HTTPRoute for IPv6 Traffic

```yaml
# HTTPRoute (Gateway API) for traffic management
apiVersion: policy.linkerd.io/v1beta2
kind: HTTPRoute
metadata:
  name: my-app-route
  namespace: default
spec:
  parentRefs:
    - name: my-service
      kind: Service
      group: core
      port: 80
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: "/api"
      backendRefs:
        - name: my-app-backend
          port: 80
          weight: 100
```

## Server and ServerAuthorization for mTLS

```yaml
# Define a Server (inbound listener)
apiVersion: policy.linkerd.io/v1beta2
kind: Server
metadata:
  name: my-app-server
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: my-app
  port: 80
  proxyProtocol: HTTP/1
---
# Allow specific clients via mTLS identity
apiVersion: policy.linkerd.io/v1beta2
kind: ServerAuthorization
metadata:
  name: my-app-authz
  namespace: default
spec:
  server:
    name: my-app-server
  client:
    meshTLS:
      serviceAccounts:
        - name: frontend
          namespace: default
```

## Observability for IPv6 Traffic

```bash
# Install Linkerd Viz extension
linkerd viz install | kubectl apply -f -
linkerd viz check

# View real-time traffic stats (works for IPv4 and IPv6)
linkerd viz stat deploy/my-app

# Output includes:
# NAME      MESHED  SUCCESS  RPS  LATENCY_P50  LATENCY_P95  LATENCY_P99
# my-app    2/2     100.00%  1.2  1ms          2ms          3ms

# View live traffic (tap)
linkerd viz tap deploy/my-app

# Check specific IPv6 source connections
linkerd viz tap deploy/my-app --from-ip "fd00::5"
```

## Linkerd Dashboard

```bash
# Open Linkerd dashboard
linkerd viz dashboard &

# The dashboard shows all traffic including IPv6
# Browse to http://localhost:50750
```

## Troubleshooting Linkerd IPv6

```bash
# Check proxy is handling IPv6 traffic
kubectl exec -c linkerd-proxy \
  $(kubectl get pod -l app=my-app -o name | head -1) \
  -- env | grep LINKERD2_PROXY

# Check proxy inbound/outbound ports
kubectl exec -c linkerd-proxy \
  $(kubectl get pod -l app=my-app -o name | head -1) \
  -- ss -6 -tlnp

# Check iptables rules for IPv6 interception
kubectl exec \
  $(kubectl get pod -l app=my-app -o name | head -1) \
  -- ip6tables -t nat -L -n

# Linkerd destination controller logs
kubectl logs -n linkerd deploy/linkerd-destination | grep -i "ipv6\|error" | tail -20

# Check if service endpoints include IPv6
kubectl get endpoints my-service -o yaml | grep -A 3 "addresses"
```

Linkerd handles IPv6 transparently on dual-stack clusters — no Linkerd-specific IPv6 configuration is needed beyond having a working dual-stack Kubernetes cluster. The Linkerd proxy intercepts both IPv4 and IPv6 traffic through iptables/ip6tables rules set up at pod startup.
