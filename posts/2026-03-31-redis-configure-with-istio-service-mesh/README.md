# How to Configure Redis with Istio Service Mesh

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Istio, Service Mesh, Kubernetes, mTLS, Networking

Description: Learn how to deploy Redis in a Kubernetes cluster with Istio service mesh, configure mTLS, traffic policies, and circuit breaking for production resilience.

---

## Why Use Istio with Redis

Istio adds service mesh capabilities to Redis running in Kubernetes without changing application code:

- **mTLS**: automatic certificate rotation and encrypted connections between services
- **Traffic management**: circuit breaking, retries, timeouts via `DestinationRule`
- **Observability**: metrics, distributed tracing, and access logs for Redis traffic
- **Authorization policies**: restrict which services can connect to Redis

## Prerequisites

A working Kubernetes cluster with Istio installed:

```bash
istioctl install --set profile=default
kubectl label namespace redis-ns istio-injection=enabled
```

## Deploying Redis with Istio Sidecar Injection

Create a Redis deployment in a namespace with Istio injection enabled:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: redis-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7.2
          ports:
            - containerPort: 6379
          args:
            - redis-server
            - --requirepass
            - $(REDIS_PASSWORD)
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-secret
                  key: password
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: redis-ns
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
      name: tcp-redis
```

The `name: tcp-redis` port name is required. Istio uses the port name to determine the protocol. Prefix it with `tcp-` for TCP passthrough.

## Enabling mTLS for Redis Traffic

Apply a `PeerAuthentication` policy to enforce mTLS in the Redis namespace:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: redis-mtls
  namespace: redis-ns
spec:
  mtls:
    mode: STRICT
```

With `STRICT` mode, all traffic to the Redis service must use mTLS. The Istio sidecar handles certificate exchange transparently - application code connects to Redis normally.

## Configuring a DestinationRule for Redis

Create a `DestinationRule` to configure traffic policies:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: redis-destination
  namespace: redis-ns
spec:
  host: redis.redis-ns.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 3s
        tcpKeepalive:
          time: 7200s
          interval: 75s
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 100
    tls:
      mode: ISTIO_MUTUAL
```

Key settings:

- `maxConnections`: limits simultaneous TCP connections to Redis
- `connectTimeout`: fail fast if Redis is unreachable
- `outlierDetection`: circuit breaker that ejects Redis from load balancing after consecutive errors
- `tls.mode: ISTIO_MUTUAL`: use Istio-managed certificates for mTLS

## AuthorizationPolicy: Restricting Access to Redis

Only allow specific services to connect to Redis:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: redis-allow-app
  namespace: redis-ns
spec:
  selector:
    matchLabels:
      app: redis
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/app-ns/sa/backend-service
              - cluster.local/ns/app-ns/sa/cache-worker
```

This allows only pods running as `backend-service` or `cache-worker` service accounts to reach Redis. All other traffic is denied.

## Observing Redis Traffic with Istio

View real-time metrics for Redis connections:

```bash
kubectl exec -n istio-system deploy/prometheus -- \
  curl -s "http://localhost:9090/api/v1/query?query=istio_tcp_connections_opened_total{destination_service_name=\"redis\"}"
```

Check Redis access logs via the Envoy sidecar:

```bash
kubectl logs -n redis-ns -l app=redis -c istio-proxy | tail -20
```

```text
[2024-01-15T10:30:00.123Z] "- - -" 0 - - - "-" 482 156 5 - "-" "-" "-" "-" "10.0.0.5:6379"
upstream_service_time=5ms
```

Use Kiali to visualize Redis traffic topology:

```bash
kubectl port-forward svc/kiali -n istio-system 20001:20001
```

Open `http://localhost:20001` and navigate to the service graph.

## Configuring Retry Policies

Add retry logic for transient Redis connection failures:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: redis-vs
  namespace: redis-ns
spec:
  hosts:
    - redis.redis-ns.svc.cluster.local
  tcp:
    - route:
        - destination:
            host: redis.redis-ns.svc.cluster.local
            port:
              number: 6379
```

Note: Istio `VirtualService` retry policies apply at the HTTP level. For TCP protocols like Redis, use `DestinationRule` outlier detection for circuit breaking.

## Egress Control for External Redis

If Redis is hosted externally (Redis Cloud, Elasticache), register it as a `ServiceEntry`:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-redis
  namespace: app-ns
spec:
  hosts:
    - redis.example.com
  ports:
    - number: 6379
      name: tcp-redis
      protocol: TCP
  location: MESH_EXTERNAL
  resolution: DNS
```

## Troubleshooting Istio-Redis Connectivity

Check if the sidecar is intercepting Redis traffic:

```bash
kubectl exec -n redis-ns deploy/redis -c istio-proxy -- \
  pilot-agent request GET clusters | grep redis
```

Verify mTLS certificate status:

```bash
istioctl authn tls-check pod/redis-pod-xyz.redis-ns redis.redis-ns.svc.cluster.local
```

```text
HOST:PORT                               STATUS    SERVER        CLIENT
redis.redis-ns.svc.cluster.local:6379  OK        STRICT        ISTIO_MUTUAL
```

## Summary

Deploying Redis with Istio provides automatic mTLS encryption, circuit breaking, and fine-grained access control without modifying Redis or your application. The key configuration elements are a TCP port name for protocol detection, a `PeerAuthentication` policy for mTLS enforcement, a `DestinationRule` for connection pooling and outlier detection, and `AuthorizationPolicy` to restrict which services can connect to Redis. Istio's observability stack then gives you metrics and access logs for Redis traffic at no additional instrumentation cost.
