# How to Set Up Kubernetes RBAC Alongside Portainer RBAC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, RBAC, Security, Access Control

Description: Configure Kubernetes native RBAC policies alongside Portainer's access control to provide layered security for cluster resources.

## Introduction

Portainer Business Edition provides team-based access control at the namespace level. However, for production environments, layering Portainer's RBAC with Kubernetes native RBAC provides defense-in-depth—ensuring that even if Portainer access is compromised, Kubernetes RBAC limits what can be done.

## How Portainer and Kubernetes RBAC Interact

When Portainer manages a Kubernetes cluster:
1. Portainer uses a service account with cluster-admin permissions
2. Portainer's team/namespace access controls what users can see in the UI
3. Kubernetes native RBAC controls what the Portainer service account can do

## Creating Kubernetes RBAC for Portainer Access

```yaml
# portainer-rbac.yml - deploy via Portainer
# Create a namespace-scoped role for developers
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: development
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods/log", "pods/exec"]
  verbs: ["get", "create"]
# Explicitly deny: no secrets access, no RBAC changes
---
# Bind the role to a service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
- kind: ServiceAccount
  name: developer-sa
  namespace: development
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
---
# Read-only role for QA/Audit
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: qa-readonly
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: []  # Explicitly no access to secrets
---
# Platform team: full cluster access except RBAC changes
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: platform-team
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]  # Can view but not modify RBAC
```

## Portainer Team to Kubernetes RBAC Mapping

```bash
# In Portainer:
# 1. Create Teams: developer-team, qa-team, platform-team
# 2. Assign namespaces: developer-team → development namespace
# 3. Set Portainer role: Standard User

# In Kubernetes:
# Create corresponding RBAC for each team's service account

# Get Portainer's service account used for kubectl access
kubectl get serviceaccount -n portainer

# Create namespace-specific service accounts
kubectl create serviceaccount developer-sa -n development
kubectl create serviceaccount qa-sa -n default

# Generate kubeconfigs for teams (Portainer integrates these)
kubectl create token developer-sa -n development --duration=8h
```

## Restricting Portainer's Own Service Account

```yaml
# Limit what Portainer can do via its service account
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: portainer-restricted
rules:
# Allow all common operations
- apiGroups: ["", "apps", "batch", "extensions"]
  resources: ["*"]
  verbs: ["*"]
# Restrict cluster-level changes
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles", "clusterrolebindings"]
  verbs: ["get", "list", "watch"]
# No access to security policies
- apiGroups: ["policy"]
  resources: ["podsecuritypolicies"]
  verbs: ["get", "list", "watch"]
```

## Audit RBAC Configurations

```bash
# Check effective permissions for a service account
kubectl auth can-i --list --as=system:serviceaccount:development:developer-sa -n development

# Check specific permission
kubectl auth can-i delete pods \
  --as=system:serviceaccount:development:developer-sa \
  -n development

# View all RoleBindings in a namespace
kubectl get rolebindings -n development -o wide
```

## Conclusion

Layering Portainer's team-based access control with Kubernetes native RBAC provides comprehensive security. Portainer handles the UI-level access and team management, while Kubernetes RBAC enforces permissions at the API level. This defense-in-depth approach ensures consistent security enforcement regardless of how users interact with the cluster.
