# How to Implement Network Segmentation in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Network Segmentation, Kubernetes, Network Policies, Security

Description: Learn how to implement network segmentation in Rancher-managed Kubernetes clusters using Projects, Network Policies, and CNI plugins to isolate workloads.

## Introduction

Network segmentation in Kubernetes limits communication between workloads, reducing the blast radius of a compromise. Rancher enhances Kubernetes network segmentation through its Project abstraction and built-in support for network policies, making it easier to enforce isolation across teams and environments.

## Rancher Projects as Isolation Boundaries

Rancher Projects group namespaces and can enforce network isolation between them. Projects provide:
- RBAC boundaries (teams see only their namespaces)
- Resource quota management
- Network isolation (with a CNI that supports Network Policies)

### Creating Isolated Projects

In Rancher:
1. Navigate to your cluster > **Projects/Namespaces**.
2. Click **Create Project**.
3. Set a name (e.g., "frontend", "backend", "databases").
4. Assign namespaces to the project.

Enable project network isolation:
```text
Project Settings > Network Isolation > Enabled
```

This automatically creates a NetworkPolicy that blocks all ingress from other projects.

## Default Deny Policy

Apply a default-deny policy to all new namespaces:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

Deploy via Rancher's **Import YAML** in the namespace.

## Allowing Specific Traffic

### Frontend to Backend

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
          podSelector:
            matchLabels:
              app: web
      ports:
        - protocol: TCP
          port: 8080
```

### Backend to Database

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-postgres
  namespace: databases
spec:
  podSelector:
    matchLabels:
      app: postgres
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: backend
      ports:
        - protocol: TCP
          port: 5432
```

### Allow DNS (Required for All Namespaces)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: backend
spec:
  podSelector: {}
  egress:
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
  policyTypes:
    - Egress
```

## Monitoring-Namespace Access

Allow Prometheus to scrape metrics from any namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: backend
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: cattle-monitoring-system
      ports:
        - protocol: TCP
          port: 9090
```

## Using Cilium for Advanced Segmentation

If using Cilium as your CNI, leverage CiliumNetworkPolicy for Layer 7 policies:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-api-calls-only
  namespace: backend
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: web
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: /api/v1/.*
              - method: POST
                path: /api/v1/.*
```

This restricts traffic to specific HTTP methods and paths.

## Validating Network Segmentation

```bash
# Test that frontend can reach backend

kubectl exec -n frontend deploy/web -- curl -s http://api.backend.svc:8080/health

# Test that frontend cannot reach database directly
kubectl exec -n frontend deploy/web -- \
  nc -zv postgres.databases.svc 5432
# Should timeout/fail
```

## Best Practices

- Enable Rancher Project Network Isolation for team-level segmentation.
- Always apply a default-deny policy and whitelist required traffic.
- Label namespaces with `kubernetes.io/metadata.name` for accurate namespace selectors.
- Test both allowed and denied paths after applying policies.
- Use Rancher's audit logs to track policy changes.

## Conclusion

Network segmentation in Rancher combines the power of Kubernetes Network Policies with Rancher's Project abstraction for a layered security model. By defining clear communication paths and denying everything else, you significantly reduce the attack surface of your workloads.
