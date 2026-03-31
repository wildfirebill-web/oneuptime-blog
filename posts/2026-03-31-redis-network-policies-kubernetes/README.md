# How to Configure Redis Network Policies in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Kubernetes, Network Policy, Security, Namespace

Description: Learn how to write Kubernetes Network Policies to restrict Redis traffic, allowing only authorized pods and namespaces to connect to your Redis instances.

---

By default, Kubernetes allows all pods to communicate with each other. This means any compromised pod can reach your Redis instance. Network Policies let you define strict ingress and egress rules to lock down Redis access.

## Prerequisites

Your cluster must have a CNI plugin that supports Network Policies, such as Calico, Cilium, or Weave Net. Vanilla flannel does not enforce Network Policies.

## Deny All Traffic First

Start with a default-deny policy in the Redis namespace, then explicitly allow only what is needed:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: redis
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

## Allow Only Specific Application Pods to Reach Redis

Allow pods labeled `app: my-api` in the `default` namespace to connect to Redis on port 6379:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-redis
  namespace: redis
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: default
      podSelector:
        matchLabels:
          app: my-api
    ports:
    - protocol: TCP
      port: 6379
```

## Allow Redis Cluster Intra-Node Communication

Redis Cluster nodes communicate on ports 6379 (data) and 16379 (cluster bus). Allow this internal traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-redis-cluster-internal
  namespace: redis
spec:
  podSelector:
    matchLabels:
      app: redis-cluster
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: redis-cluster
    ports:
    - protocol: TCP
      port: 6379
    - protocol: TCP
      port: 16379
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: redis-cluster
    ports:
    - protocol: TCP
      port: 6379
    - protocol: TCP
      port: 16379
```

## Allow Monitoring Access (Prometheus)

Let Prometheus scrape the Redis exporter on port 9121:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: redis
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
      podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9121
```

## Allow DNS Egress

Redis pods need DNS resolution. Allow egress to kube-dns:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: redis
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

## Verifying Network Policies

Test that unauthorized pods cannot reach Redis:

```bash
# Deploy a test pod in an unauthorized namespace
kubectl run test-pod --image=redis:7.2-alpine --restart=Never -n default -- sleep 3600

# Try to connect - this should fail
kubectl exec -it test-pod -n default -- redis-cli -h redis.redis.svc.cluster.local ping
```

Expected output:

```text
Could not connect to Redis at redis.redis.svc.cluster.local:6379: Connection timed out
```

## Summary

Kubernetes Network Policies are your first line of defense for Redis security in a cluster. Start with a default-deny-all policy, then selectively allow traffic from your application pods, monitoring systems, and cluster-internal communication. Always label your namespaces to enable namespace selectors in policies, and verify enforcement with a test pod from an unauthorized namespace.
