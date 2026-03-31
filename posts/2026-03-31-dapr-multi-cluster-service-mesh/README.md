# How to Use Dapr with Multi-Cluster Service Mesh

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Multi-Cluster, Service Mesh, Istio, Kubernetes, Federation

Description: Learn how to configure Dapr service invocation and pub/sub across multiple Kubernetes clusters using a multi-cluster service mesh like Istio or Skupper.

---

## Multi-Cluster Dapr Architecture

Dapr is namespace-scoped by default. For multi-cluster deployments, you need a mechanism to route service invocation and pub/sub traffic between clusters. Two common approaches are:

1. **Istio Multi-Cluster** - Extend service mesh across clusters with unified mTLS
2. **Skupper** - Layer 7 network connectivity without a service mesh
3. **Shared message broker** - Use a cross-cluster Kafka or Redis for pub/sub

## Option 1: Istio Multi-Primary Multi-Network

Configure two clusters with Istio in multi-primary mode so Dapr sidecars can discover services in either cluster:

```bash
# Cluster 1 setup
istioctl install -f cluster1.yaml --context cluster1

# Cluster 2 setup
istioctl install -f cluster2.yaml --context cluster2

# Exchange root certificates
kubectl get secret istio-ca-secret -n istio-system \
  --context cluster1 -o jsonpath='{.data.ca-cert\.pem}' \
  | base64 -d > cluster1-ca.pem

# Create remote secret on cluster2
istioctl create-remote-secret \
  --context=cluster1 \
  --name=cluster1 | kubectl apply --context=cluster2 -f -
```

## Dapr Service Invocation Across Clusters

With Istio multi-cluster, services in cluster2 are accessible via their `ServiceEntry`:

```yaml
# Register cluster2 service in cluster1
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: inventory-service-remote
  namespace: production
spec:
  hosts:
    - inventory-service.production.svc.cluster.remote
  ports:
    - number: 80
      name: http
      protocol: HTTP
  resolution: DNS
  location: MESH_INTERNAL
```

Dapr can invoke this remote service using name resolution:

```go
// This call is routed across clusters via Istio
resp, err := client.InvokeMethod(ctx,
    "inventory-service",
    "check-stock",
    "GET")
```

## Option 2: Cross-Cluster Pub/Sub with Shared Kafka

The simplest cross-cluster Dapr integration uses a shared Kafka cluster:

```yaml
# Both clusters use the same Kafka component config
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: cross-cluster-pubsub
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: kafka.shared-infra.example.com:9092
    - name: consumerGroup
      value: cluster1-consumers    # Different per cluster
    - name: authRequired
      value: "true"
    - name: saslUsername
      secretKeyRef:
        name: kafka-secret
        key: username
```

## Option 3: Skupper for Layer 7 Cross-Cluster Connectivity

Skupper creates an application-layer network between clusters without requiring a service mesh:

```bash
# Install Skupper in cluster1
skupper init --context cluster1 --site-name cluster1

# Install Skupper in cluster2
skupper init --context cluster2 --site-name cluster2

# Link the clusters
skupper token create cluster1-link.yaml --context cluster1
skupper link create cluster1-link.yaml --context cluster2

# Expose Dapr-enabled service across clusters
skupper expose deployment inventory-service \
  --context cluster1 \
  --port 80
```

## Monitoring Cross-Cluster Dapr Traffic

Track cross-cluster service invocation with Prometheus federation:

```yaml
# prometheus.yaml in cluster1 - federate cluster2 metrics
scrape_configs:
  - job_name: 'dapr-cluster2'
    honor_labels: true
    metrics_path: '/federate'
    params:
      match[]:
        - '{job="dapr-sidecar"}'
    static_configs:
      - targets:
          - prometheus.cluster2.example.com:9090
```

## Cross-Cluster Resiliency

Configure resiliency policies for cross-cluster calls:

```yaml
spec:
  policies:
    retries:
      cross-cluster:
        policy: exponential
        initialInterval: 1s
        maxRetries: 5
        maxInterval: 30s
    circuitBreakers:
      remote-cluster-cb:
        trip: consecutiveFailures >= 3
        timeout: 60s    # Longer timeout for network partitions
  targets:
    apps:
      inventory-service:
        retry: cross-cluster
        circuitBreaker: remote-cluster-cb
```

## Summary

Multi-cluster Dapr deployments use either Istio multi-primary mode for transparent cross-cluster service invocation, a shared message broker (Kafka/Redis) for cross-cluster pub/sub, or Skupper for application-layer connectivity. Configure longer timeouts and more aggressive circuit breakers for cross-cluster calls to handle the higher latency and network partition scenarios inherent in multi-cluster architectures.
