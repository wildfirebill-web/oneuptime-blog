# How to Prevent Dapr Control Plane Pod Eviction on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, Eviction, Pod Disruption Budget, Reliability

Description: Prevent Dapr control plane pods from being evicted during node pressure or voluntary disruptions using PodDisruptionBudgets, priority classes, and resource guarantees.

---

## Why Eviction Is a Problem

Dapr control plane pods can be evicted when nodes experience memory pressure or when cluster administrators drain nodes for maintenance. Evicting the sentry disrupts mTLS certificate issuance, and evicting the operator stops component reconciliation.

## Setting Resource Requests to Prevent Pressure-Based Eviction

Pods with resource requests set to equal their limits get Guaranteed QoS class and are the last to be evicted:

```yaml
# dapr-qos-values.yaml
dapr_operator:
  resources:
    requests:
      cpu: "250m"
      memory: "256Mi"
    limits:
      cpu: "250m"
      memory: "256Mi"

dapr_sentry:
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "100m"
      memory: "128Mi"

dapr_sidecar_injector:
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "100m"
      memory: "128Mi"
```

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  -f dapr-qos-values.yaml
```

## Creating PodDisruptionBudgets

PodDisruptionBudgets prevent voluntary disruptions (node drains) from taking down too many replicas:

```yaml
# dapr-pdbs.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: dapr-operator-pdb
  namespace: dapr-system
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: dapr-operator
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: dapr-sentry-pdb
  namespace: dapr-system
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: dapr-sentry
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: dapr-sidecar-injector-pdb
  namespace: dapr-system
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: dapr-sidecar-injector
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: dapr-placement-pdb
  namespace: dapr-system
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: dapr-placement-server
```

```bash
kubectl apply -f dapr-pdbs.yaml
kubectl get pdb -n dapr-system
```

## Setting Priority Class to Prevent Preemption

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set dapr_operator.priorityClassName=system-cluster-critical \
  --set dapr_sentry.priorityClassName=system-cluster-critical \
  --set dapr_sidecar_injector.priorityClassName=system-cluster-critical \
  --set dapr_placement.priorityClassName=system-cluster-critical
```

## Verifying Anti-Eviction Settings

```bash
# Check QoS class of control plane pods
kubectl get pods -n dapr-system -o custom-columns=\
"NAME:.metadata.name,QOS:.status.qosClass"

# Check PDB status
kubectl get pdb -n dapr-system
# DISRUPTIONS ALLOWED should be > 0 only when you have enough replicas

# Simulate a drain and watch PDB protect pods
kubectl drain <node-name> --ignore-daemonsets --dry-run=client
```

## Summary

Preventing Dapr control plane eviction requires a combination of Guaranteed QoS (equal requests and limits), PodDisruptionBudgets to protect against voluntary disruptions, and high-priority PriorityClasses to resist preemption under node resource pressure. Together these three mechanisms ensure Dapr's infrastructure remains available during both routine maintenance and unexpected resource spikes.
