# How to Configure Kubernetes Cluster Security Settings in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Security, RBAC, Container Security

Description: Learn how to configure security settings for a Kubernetes cluster managed through Portainer to restrict user capabilities.

## Overview

Portainer exposes cluster-level security settings that restrict what non-admin users can do within a Kubernetes environment. These settings act as guardrails on top of standard Kubernetes RBAC.

## Accessing Cluster Security Settings

1. In Portainer, select your Kubernetes environment.
2. Go to **Cluster > Setup** or **Cluster > Security**.
3. Configure the available security options.

## Available Security Settings

### Allow Users to Use the Load Balancer Feature

Disabling this prevents regular users from creating LoadBalancer services, which could incur unexpected cloud costs.

### Allow Users to Use External IPs for Services

Restricts users from specifying `externalIPs` in service definitions, which can cause security issues.

### Restrict Default Namespace Usage

Prevents workloads from being deployed to the `default` namespace, enforcing namespace isolation.

### Enable Node Selector for Workloads

Forces users to specify a node selector for all workloads, ensuring proper placement.

## Configuring Pod Security Standards

Kubernetes v1.25+ uses Pod Security Standards (PSS) instead of Pod Security Policies. Configure them via namespace labels:

```bash
# Apply the "restricted" security standard to a namespace
kubectl label namespace my-namespace \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Apply "baseline" policy (less strict)
kubectl label namespace my-namespace \
  pod-security.kubernetes.io/enforce=baseline
```

## Configuring RBAC for Portainer Users

Create custom Kubernetes roles for Portainer-managed teams:

```yaml
# developer-role.yaml - Read-only access for developers
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: my-namespace
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: my-namespace
subjects:
  - kind: User
    name: developer@company.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

## Restricting Privileged Containers

Prevent privileged containers via a NetworkPolicy or OPA/Gatekeeper constraint:

```yaml
# OPA Gatekeeper constraint to block privileged containers
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: psp-privileged-container
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

## Auditing Security Settings

```bash
# Check what security-related admission webhooks are active
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# List cluster role bindings to audit access
kubectl get clusterrolebindings -o wide | grep -v "system:"
```

## Conclusion

Portainer's cluster security settings combined with Kubernetes RBAC give you a layered security model. Always apply the principle of least privilege — give users only the permissions they need for their role.
