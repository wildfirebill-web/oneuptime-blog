# How to Use Dapr with Kubernetes Topology Spread Constraints

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, Topology, Availability Zone, Scheduling

Description: Use Kubernetes Topology Spread Constraints to distribute Dapr-enabled pods across availability zones and nodes for improved fault tolerance and resilience.

---

## Overview

Topology Spread Constraints let you control how Kubernetes schedules pods across failure domains like availability zones or nodes. For Dapr services, spreading pods across zones ensures that a single zone failure does not take down an entire service.

## Basic Zone Spread Constraint

Add topology spread constraints to your Dapr deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: default
spec:
  replicas: 6
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "3000"
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: order-service
      containers:
      - name: order-service
        image: myregistry/order-service:latest
```

## Spreading Across Nodes

Spread pods across individual nodes to avoid single-node failures:

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: payment-service
```

Using `ScheduleAnyway` is softer - Kubernetes will try to spread but won't block scheduling if it cannot achieve the ideal distribution.

## Combining Zone and Node Constraints

Use multiple constraints for defense-in-depth:

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: inventory-service
- maxSkew: 2
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: inventory-service
```

## Verifying Pod Distribution

After deployment, verify pods are spread across zones:

```bash
kubectl get pods -l app=order-service -o wide
# Check the NODE column to confirm different nodes/zones

# Get zone labels
kubectl get nodes -L topology.kubernetes.io/zone
```

## Impact on Dapr Actor Placement

When using Dapr actors, the placement service distributes actor instances across pods. Combining topology spread constraints with at least 3 replicas ensures actors are not all concentrated in one zone:

```bash
# Check actor distribution via Dapr dashboard
kubectl port-forward svc/dapr-dashboard 8080:8080 -n dapr-system
# Open http://localhost:8080 and check actor placements
```

## Dapr Control Plane Spread

Apply similar constraints to Dapr's own operator and sidecar injector:

```bash
kubectl patch deployment dapr-operator -n dapr-system --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/topologySpreadConstraints","value":[{"maxSkew":1,"topologyKey":"topology.kubernetes.io/zone","whenUnsatisfiable":"ScheduleAnyway","labelSelector":{"matchLabels":{"app":"dapr-operator"}}}]}]'
```

## Summary

Kubernetes Topology Spread Constraints ensure Dapr service pods are distributed across availability zones and nodes, reducing the blast radius of infrastructure failures. By combining zone-level hard constraints with node-level soft constraints, you achieve both fault isolation and efficient resource utilization for your Dapr-enabled microservices.
