# How to Configure Kubernetes Cluster Security Settings in Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Security, RBAC, DevOps

Description: Learn how to configure security settings for Kubernetes clusters in Portainer including RBAC, network policies, and access controls.

## Introduction

Securing a Kubernetes cluster managed by Portainer involves configuring proper RBAC, network policies, and Portainer-specific access controls. This guide covers the security settings available in Portainer for Kubernetes clusters and best practices for hardening your cluster management.

## Prerequisites

- Portainer BE with Kubernetes environment
- Cluster-admin access
- Understanding of Kubernetes RBAC concepts

## Step 1: Configure Portainer Kubernetes Security Settings

In Portainer, navigate to your Kubernetes environment:

1. Click on the environment
2. Go to **Settings → Cluster** (or **Environment Settings**)

Available security settings:

```text
Cluster settings:
  [x] Enable resource over-commit   - Allow workloads to request more than available
  [ ] Restrict default namespace   - Prevent deploying to default namespace
  [x] Use private registries       - Require registry credentials for image pulls
  [x] Allow web editor             - Let users edit YAML directly
  [ ] Note: Disabling web editor enforces form-based deployments only
```

## Step 2: Configure Namespace Access Control

In Portainer BE, assign namespaces to teams:

1. Go to **Namespaces** in the Kubernetes environment
2. Click on a namespace
3. Under **Access control**, assign to teams:

```text
Namespace: production
Access control:
  Team: backend-team    → Read/Write
  Team: frontend-team   → Read only
  Team: devops-team     → Admin
```

This prevents teams from accessing each other's namespaces.

## Step 3: Apply RBAC with Portainer

Portainer creates Kubernetes RBAC resources when you configure team access. The resulting RBAC:

```yaml
# Portainer creates these resources automatically

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: portainer-rw
  namespace: production
rules:
  - apiGroups: ["", "apps", "batch", "extensions"]
    resources: ["*"]
    verbs: ["get", "list", "create", "update", "delete", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: portainer-rw-backend-team
  namespace: production
subjects:
  - kind: Group
    name: backend-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: portainer-rw
  apiGroup: rbac.authorization.k8s.io
```

## Step 4: Implement Network Policies

Apply network policies to restrict inter-namespace communication:

```yaml
# deny-all-ingress.yaml - Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}    # Applies to all pods
  policyTypes:
    - Ingress
```

```yaml
# allow-frontend-to-api.yaml - Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api          # Apply to API pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend   # Only allow from frontend pods
      ports:
        - protocol: TCP
          port: 8080
```

Deploy via Portainer's YAML editor under **Applications → Add application → From YAML**.

## Step 5: Configure Pod Security

```yaml
# Pod Security Standards (Kubernetes 1.25+)
# Apply to a namespace via labels
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

For specific pod restrictions:

```yaml
spec:
  containers:
    - name: app
      image: myapp:latest
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

## Step 6: Audit Portainer Actions

Portainer logs all user actions. View audit logs:

1. Go to **Settings → Audit logs** (BE feature)
2. Filter by user, action, or time range

```text
2024-01-15 10:00:01  admin         CREATE   deployment  production/myapp
2024-01-15 10:05:32  john.doe      DELETE   pod         staging/api-xxxx
2024-01-15 10:10:44  jane.smith    UPDATE   configmap   development/app-config
```

## Step 7: Restrict Service Account Tokens

Disable automatic service account token mounting:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: production
automountServiceAccountToken: false   # Disable auto-mounting
```

Or at the pod level:

```yaml
spec:
  automountServiceAccountToken: false
  serviceAccountName: myapp
```

## Step 8: Configure RBAC for Portainer Service Account

Limit what the Portainer service account can do:

```yaml
# Limited RBAC for read-only Portainer access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: portainer-read-only
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints", "namespaces", "nodes", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets", "replicasets"]
    verbs: ["get", "list", "watch"]
```

## Conclusion

Kubernetes security in Portainer operates on two levels: Portainer-native access control (team-based namespace access) and Kubernetes-native security (RBAC, NetworkPolicy, Pod Security). Use Portainer's team access features to control who can deploy to which namespaces, and implement Kubernetes RBAC and NetworkPolicy for fine-grained resource access control. Regular audit log reviews help detect unauthorized or accidental changes.
