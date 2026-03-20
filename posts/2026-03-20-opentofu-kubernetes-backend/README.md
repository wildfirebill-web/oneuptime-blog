# How to Configure the Kubernetes Backend in OpenTofu - Opentofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Backend, Kubernetes

Description: Learn how to configure the Kubernetes backend in OpenTofu to store state as Kubernetes Secrets with built-in locking using leases.

## Introduction

The Kubernetes backend stores OpenTofu state as Kubernetes Secrets and uses Kubernetes Lease objects for state locking. It is a natural fit for teams running infrastructure automation from within Kubernetes clusters, such as operators or CI/CD runners.

## Basic Configuration

```hcl
terraform {
  backend "kubernetes" {
    secret_suffix    = "production"
    load_config_file = true  # Use local kubeconfig
  }
}
```

State is stored as: `Secret/tfstate-default-production` in the default namespace.

## In-Cluster Configuration

When OpenTofu runs inside a Kubernetes pod, use in-cluster authentication:

```hcl
terraform {
  backend "kubernetes" {
    secret_suffix     = "production"
    in_cluster_config = true
    namespace         = "terraform"
  }
}
```

The pod's service account is used automatically.

## Explicit kubeconfig

```hcl
terraform {
  backend "kubernetes" {
    secret_suffix   = "production"
    config_path     = "/home/runner/.kube/config"
    config_context  = "my-cluster"
    namespace       = "terraform"
  }
}
```

## Service Account for In-Cluster Use

```yaml
# Create a dedicated namespace and service account

apiVersion: v1
kind: Namespace
metadata:
  name: terraform
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tofu-runner
  namespace: terraform
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: terraform
  name: tofu-state-manager
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "create", "update", "delete", "list"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tofu-state-binding
  namespace: terraform
subjects:
  - kind: ServiceAccount
    name: tofu-runner
    namespace: terraform
roleRef:
  kind: Role
  name: tofu-state-manager
  apiGroup: rbac.authorization.k8s.io
```

## Multiple Configurations per Namespace

Use different `secret_suffix` values to separate configurations:

```hcl
# networking/backend.tf
terraform {
  backend "kubernetes" {
    secret_suffix = "networking"
    namespace     = "terraform"
    in_cluster_config = true
  }
}

# applications/backend.tf
terraform {
  backend "kubernetes" {
    secret_suffix = "applications"
    namespace     = "terraform"
    in_cluster_config = true
  }
}
```

This creates:
- `Secret/tfstate-default-networking`
- `Secret/tfstate-default-applications`

## Inspecting the Stored State

```bash
# List state secrets
kubectl get secrets -n terraform -l app.kubernetes.io/managed-by=terraform

# Read the raw state (base64 encoded)
kubectl get secret tfstate-default-production -n terraform -o jsonpath='{.data.tfstate}' | base64 -d | jq .
```

## Workspace Isolation

```bash
# Create workspaces - stored as separate secrets
tofu workspace new staging
tofu workspace new production

# Results in secrets:
# tfstate-default-staging
# tfstate-default-production
```

## Configuration via Environment Variables

```bash
export KUBE_CONFIG_PATH="$HOME/.kube/config"
export KUBE_CONTEXT="my-cluster"

tofu init -backend-config="secret_suffix=production" \
          -backend-config="namespace=terraform"
```

## Conclusion

The Kubernetes backend is ideal for teams running OpenTofu from within Kubernetes. State is stored as encrypted-at-rest Kubernetes Secrets (encryption depends on your cluster configuration), and locking uses Kubernetes Lease objects. Use dedicated namespaces and RBAC roles to restrict access to state secrets.
