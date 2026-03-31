# How to Configure Multi-Manager High Availability in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Manager, High Availability, Kubernetes, Dashboard

Description: Learn how to configure multiple Ceph manager daemons in Rook for high availability, covering active-standby behavior, module management, and failover configuration.

---

## Overview

The Ceph Manager (MGR) daemon provides monitoring data, handles the Ceph Dashboard, and runs pluggable modules like the Prometheus exporter, PG autoscaler, and balancer. Running two managers in an active-standby configuration ensures these functions remain available if the active manager crashes. This guide covers configuring multi-manager HA in Rook.

## Active-Standby Model

Ceph MGR uses an active-standby pattern:
- Only one manager is **active** at a time
- All other managers are **standby**, ready to take over
- Failover is automatic and typically completes in seconds

```bash
# Check current manager status
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  ceph mgr stat

# Output:
# {
#   "epoch": 42,
#   "available": true,
#   "active_name": "a",
#   "num_standby": 1
# }
```

## Configuring Manager Count in Rook

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  mgr:
    count: 2                        # Active + 1 standby
    allowMultiplePerNode: false     # One manager per node
    modules:
    - name: pg_autoscaler
      enabled: true
    - name: prometheus
      enabled: true
    - name: balancer
      enabled: true
  placement:
    mgr:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values: [rook-ceph-mgr]
          topologyKey: kubernetes.io/hostname
```

```bash
kubectl apply -f cluster.yaml

# Verify manager pods
kubectl get pods -n rook-ceph -l app=rook-ceph-mgr
```

## Manager Resources

```yaml
spec:
  resources:
    mgr:
      limits:
        cpu: "2"
        memory: "2Gi"
      requests:
        cpu: "500m"
        memory: "512Mi"
```

The active manager's memory usage grows with cluster size due to collected stats. Scale accordingly.

## Managing Modules

```bash
# List all available modules
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  ceph mgr module ls

# Enable a module
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  ceph mgr module enable prometheus

# Disable a module
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  ceph mgr module disable pg_autoscaler

# Check module status
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  ceph mgr module ls | grep -E "enabled_modules|disabled_modules"
```

## Forcing a Manager Failover

You can manually trigger failover to test or to force a new active manager:

```bash
# Force the active manager to fail over
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  ceph mgr fail

# Watch the new manager become active
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  ceph -w | grep "mgr"
```

## Accessing the Ceph Dashboard with HA

When using 2 managers, both run the Dashboard, but only the active one serves requests. Rook creates a Kubernetes Service that tracks the active manager:

```bash
# Get the dashboard service
kubectl get svc -n rook-ceph rook-ceph-mgr-dashboard

# Port-forward to the active dashboard
kubectl port-forward svc/rook-ceph-mgr-dashboard 8443:8443 -n rook-ceph
```

After failover, the Service automatically routes to the new active manager within seconds.

## Setting Up an Ingress for the Dashboard

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ceph-dashboard
  namespace: rook-ceph
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: ceph.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: rook-ceph-mgr-dashboard
            port:
              number: 8443
```

## Monitoring Manager Health

```bash
# Check if managers are healthy
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  ceph health detail | grep mgr

# Get manager performance stats
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  ceph mgr dump
```

## Summary

Multi-manager HA in Rook-Ceph provides automatic failover for the Dashboard, Prometheus exporter, PG autoscaler, and all other MGR modules. Configure 2 managers with strict anti-affinity rules, ensure the active manager has sufficient memory to handle cluster stats, and test failover regularly with `ceph mgr fail`. The Kubernetes Service for the Dashboard automatically follows the active manager, so no client reconfiguration is needed after a failover event.
