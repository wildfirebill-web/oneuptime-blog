# How to Configure Redis with Linkerd Service Mesh

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Linkerd, Service Mesh, Kubernetes, mTLS

Description: Learn how to configure Redis with Linkerd service mesh - covering sidecar injection, TCP traffic handling, mTLS for Redis connections, and observability via Linkerd dashboard.

---

Linkerd provides automatic mTLS, observability, and traffic management for Kubernetes workloads. Configuring Redis behind Linkerd adds encrypted connections and golden metrics without modifying application code.

## Install Linkerd

```bash
# Install the Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin

# Install Linkerd on your cluster
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# Verify installation
linkerd check
```

## Deploy Redis with Linkerd Injection

Add the Linkerd annotation to your Redis namespace or deployment:

```bash
# Annotate namespace for automatic injection
kubectl annotate namespace redis-ns linkerd.io/inject=enabled
```

Or annotate the deployment directly:

```yaml
# redis-deployment.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: redis-ns
  annotations:
    linkerd.io/inject: enabled
spec:
  selector:
    matchLabels:
      app: redis
  serviceName: redis
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
      annotations:
        linkerd.io/inject: enabled
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
          resources:
            limits:
              memory: "1Gi"
              cpu: "500m"
```

```bash
kubectl apply -f redis-deployment.yaml
```

## Configure TCP Traffic Handling

Linkerd handles TCP traffic (which Redis uses) but needs guidance on protocol detection. For Redis ports, configure opaque ports:

```bash
# Tell Linkerd to treat port 6379 as opaque TCP (no protocol detection)
kubectl annotate service redis \
  config.linkerd.io/opaque-ports="6379"
```

This prevents Linkerd from attempting HTTP/2 protocol detection on the Redis port.

## Verify mTLS is Active

Check that Linkerd is encrypting Redis connections:

```bash
# Check mTLS status between pods
linkerd viz tap deploy/redis-client -n app-ns \
  --to deploy/redis -n redis-ns

# View the mTLS status in Linkerd dashboard
linkerd viz dashboard &
```

In the Linkerd dashboard, navigate to the Redis namespace. Connections with the lock icon are mTLS encrypted.

## View Redis Traffic Metrics

Linkerd automatically collects golden metrics for TCP connections to Redis:

```bash
# View TCP traffic stats for Redis
linkerd viz stat -n redis-ns deploy/redis

# Output shows:
# NAME    MESHED  SUCCESS   RPS   LATENCY_P50  LATENCY_P95  LATENCY_P99
# redis   1/1     100.00%   45    2ms          8ms          15ms
```

## Set Up Linkerd ServiceProfiles for Circuit Breaking

While Linkerd's ServiceProfiles are primarily for HTTP, you can define retry budgets for Redis client deployments:

```yaml
# ServiceProfile for the Redis client service
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: redis.redis-ns.svc.cluster.local
  namespace: app-ns
spec:
  routes:
    - name: redis-connection
      condition:
        method: POST
        pathRegex: ".*"
      isRetryable: false  # Don't retry Redis writes
```

## Monitor with Prometheus

Linkerd exposes metrics in Prometheus format:

```bash
# Port forward to Prometheus
kubectl port-forward -n linkerd-viz svc/prometheus 9090:9090

# Query Redis TCP connection metrics
# metric: tcp_open_connections{direction="outbound", dst_namespace="redis-ns"}
```

## Summary

Linkerd adds mTLS encryption, traffic observability, and circuit breaking to Redis connections without any application code changes. The key configuration steps are enabling sidecar injection on Redis pods and annotating port 6379 as opaque to prevent protocol detection errors. Linkerd's dashboard and Prometheus metrics provide TCP-level visibility into Redis connection counts, latency percentiles, and success rates.
