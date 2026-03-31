# How to Scale the Dapr Placement Service

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Placement, Scaling, Kubernetes, High Availability

Description: Learn how to scale the Dapr placement service for high availability and production workloads, including replica counts, resource tuning, and monitoring.

---

## Overview

The Dapr placement service is the central coordinator for actor routing. In production, it must be highly available and performant enough to handle registration events from all application sidecars. This post covers how to scale the placement service and tune it for large actor deployments.

## Default Installation

By default, Dapr installs the placement service as a single-replica StatefulSet. This is fine for development but provides no fault tolerance.

```bash
# Check current placement replicas
kubectl get statefulset -n dapr-system dapr-placement-server
```

## Scaling to High Availability

Scale the placement service to 3 replicas for single-zone HA, or 5 replicas for multi-zone deployments:

```bash
# Scale to 3 replicas
kubectl scale statefulset dapr-placement-server -n dapr-system --replicas=3
```

Or configure during Helm installation:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set dapr_placement.replicaCount=3 \
  --set dapr_placement.ha.enabled=true
```

## Resource Configuration

For large clusters with many actors, tune CPU and memory limits:

```yaml
# Helm values for placement service resources
dapr_placement:
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
```

## Pod Disruption Budget

Prevent simultaneous eviction of all placement pods during cluster maintenance:

```yaml
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

## Spreading Across Zones

Use topology spread constraints to distribute placement pods across availability zones:

```yaml
# Add to placement StatefulSet spec
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: dapr-placement-server
```

## Monitoring Placement Service Load

```bash
# Check placement service metrics
curl http://dapr-placement-server-0.dapr-system:9090/metrics | grep -E "placement|raft"

# Key metrics:
# dapr_placement_runtimehosts_total - number of registered sidecars
# dapr_placement_actortypes_total - number of registered actor types
```

```bash
# Watch for placement errors in application sidecars
kubectl logs -l app=my-actor-app -c daprd | grep -i "placement.*error\|failed.*placement"
```

## Replication Factor Tuning

The `replicationFactor` controls how many virtual nodes each host claims on the hash ring. Higher values improve distribution uniformity but increase placement table size:

```bash
# Set replication factor during Helm install
helm install dapr dapr/dapr \
  --set dapr_placement.replicationFactor=100
```

## Summary

Scale the Dapr placement service to at least 3 replicas in production to provide Raft consensus fault tolerance. Use Pod Disruption Budgets and topology spread constraints to maintain availability during maintenance windows and zone failures. Monitor the placement metrics to detect registration bottlenecks and adjust resource limits as your actor workload grows.
