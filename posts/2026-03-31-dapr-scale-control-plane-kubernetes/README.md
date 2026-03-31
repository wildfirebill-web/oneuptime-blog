# How to Scale Dapr Control Plane on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, Scaling, High Availability, Control Plane

Description: Scale Dapr control plane components for high availability by adjusting replica counts for the operator, sentry, sidecar injector, and placement service.

---

## Dapr Control Plane Components

The Dapr control plane consists of four main components:

| Component | Default Replicas | HA Replicas |
|---|---|---|
| dapr-operator | 1 | 2-3 |
| dapr-sentry | 1 | 2-3 |
| dapr-sidecar-injector | 1 | 2-3 |
| dapr-placement-server | 1 | 3 (Raft) |

## Enabling HA Mode with Helm

The easiest way to scale all control plane components:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set global.ha.enabled=true \
  --set global.ha.replicaCount=3 \
  --wait
```

## Scaling Individual Components

For fine-grained control:

```bash
# Scale the operator
kubectl scale deployment dapr-operator -n dapr-system --replicas=2

# Scale sentry
kubectl scale deployment dapr-sentry -n dapr-system --replicas=2

# Scale sidecar injector
kubectl scale deployment dapr-sidecar-injector -n dapr-system --replicas=2

# Scale placement (requires odd number for Raft consensus)
kubectl scale statefulset dapr-placement-server -n dapr-system --replicas=3
```

## Verifying HA State

```bash
# Check all control plane pods
kubectl get pods -n dapr-system -o wide

# Verify placement Raft leader election
kubectl logs -n dapr-system dapr-placement-server-0 | grep -i "leader\|raft\|elected"

# Check placement server cluster health
kubectl exec -n dapr-system dapr-placement-server-0 -- \
  wget -O- http://localhost:8080/healthz
```

## Setting Anti-Affinity Rules

Prevent multiple replicas from running on the same node:

```yaml
# dapr-ha-values.yaml
dapr_operator:
  replicaCount: 2
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - dapr-operator
          topologyKey: kubernetes.io/hostname
```

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  -f dapr-ha-values.yaml
```

## Monitoring Control Plane Under Load

```bash
# Check resource usage
kubectl top pods -n dapr-system

# Check operator queue depth (if metrics enabled)
kubectl port-forward -n dapr-system svc/dapr-operator 9090:9090
curl http://localhost:9090/metrics | grep dapr_operator
```

## Summary

Scaling the Dapr control plane to HA mode uses `global.ha.enabled=true` in Helm, which sets 3 replicas for all components and configures the placement server as a 3-node Raft cluster. Use pod anti-affinity rules to spread replicas across nodes for true fault tolerance. Monitor placement server logs to verify Raft leader election after scaling.
