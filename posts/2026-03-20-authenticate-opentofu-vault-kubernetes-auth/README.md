# How to Authenticate OpenTofu with Vault Using Kubernetes Auth

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Vault, Kubernetes Auth, Service Account, Authentication

Description: Learn how to configure OpenTofu to authenticate with HashiCorp Vault using the Kubernetes auth method, enabling pods and CI/CD jobs to authenticate using service account tokens.

## Introduction

Vault's Kubernetes auth method uses Kubernetes service account JWT tokens for authentication. OpenTofu running in Kubernetes-based CI/CD systems (Argo CD, Tekton, GitLab Runners) can authenticate with Vault using the pod's mounted service account token — no separate credentials required.

## Configuring Kubernetes Auth in Vault

```hcl
# Enable Kubernetes auth method
resource "vault_auth_backend" "kubernetes" {
  type = "kubernetes"
  path = "kubernetes"
}

# Configure with the Kubernetes cluster's CA and host
resource "vault_kubernetes_auth_backend_config" "config" {
  backend            = vault_auth_backend.kubernetes.path
  kubernetes_host    = "https://kubernetes.default.svc.cluster.local"
  # Vault running inside the cluster can use:
  kubernetes_ca_cert = file("${path.module}/k8s-ca.crt")
  # Or let Vault discover it from the token reviewer's service account
  disable_iss_validation = true
}

# Create a role for the CI/CD service account
resource "vault_kubernetes_auth_backend_role" "opentofu_cicd" {
  backend                          = vault_auth_backend.kubernetes.path
  role_name                        = "opentofu-cicd"
  bound_service_account_names      = ["opentofu-runner"]
  bound_service_account_namespaces = ["ci-cd", "argocd"]
  token_policies                   = ["opentofu-policy"]
  token_ttl                        = 3600
  token_max_ttl                    = 14400
  audience                         = "vault"
}
```

## Creating the Kubernetes Service Account

```hcl
resource "kubernetes_service_account" "opentofu_runner" {
  metadata {
    name      = "opentofu-runner"
    namespace = "ci-cd"
    annotations = {
      "vault.hashicorp.com/role" = "opentofu-cicd"
    }
  }
}

# RBAC - only needs to read its own token (for Kubernetes 1.24+)
resource "kubernetes_cluster_role_binding" "token_review" {
  metadata {
    name = "opentofu-runner-token-review"
  }
  role_ref {
    api_group = "rbac.authorization.k8s.io"
    kind      = "ClusterRole"
    name      = "system:auth-delegator"
  }
  subject {
    kind      = "ServiceAccount"
    name      = kubernetes_service_account.opentofu_runner.metadata[0].name
    namespace = "ci-cd"
  }
}
```

## OpenTofu Provider Configuration

```hcl
# provider.tf - running inside Kubernetes
provider "vault" {
  address = "http://vault.vault.svc.cluster.local:8200"

  auth_login_kubernetes {
    role = "opentofu-cicd"
    # JWT token path (auto-mounted by Kubernetes)
    jwt  = file("/var/run/secrets/kubernetes.io/serviceaccount/token")
    # Or for projected service account tokens:
    # jwt  = file("/var/run/secrets/vault/token")
  }
}
```

## Tekton Pipeline Configuration

```yaml
# tekton/pipeline-run.yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: opentofu-apply
spec:
  serviceAccountName: opentofu-runner
  taskSpec:
    steps:
      - name: tofu-apply
        image: ghcr.io/opentofu/opentofu:latest
        script: |
          tofu init
          tofu apply -auto-approve
        env:
          - name: VAULT_ADDR
            value: "http://vault.vault.svc.cluster.local:8200"
          # VAULT_TOKEN obtained by Vault agent sidecar or auth_login_kubernetes
```

## Argo CD Integration with Vault Agent

```yaml
# argocd/application.yaml - using vault-agent-injector annotations
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opentofu-runner
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "opentofu-cicd"
        vault.hashicorp.com/agent-inject-secret-aws: "aws/creds/opentofu-role"
        vault.hashicorp.com/agent-inject-template-aws: |
          {{- with secret "aws/creds/opentofu-role" -}}
          export AWS_ACCESS_KEY_ID="{{ .Data.access_key }}"
          export AWS_SECRET_ACCESS_KEY="{{ .Data.secret_key }}"
          export AWS_SESSION_TOKEN="{{ .Data.security_token }}"
          {{- end }}
```

## Projected Service Account Tokens (Kubernetes 1.21+)

```yaml
# pod.yaml - mount a projected token with vault audience
volumes:
  - name: vault-token
    projected:
      sources:
        - serviceAccountToken:
            audience: vault
            expirationSeconds: 7200
            path: token
```

```hcl
provider "vault" {
  address = "http://vault.vault.svc:8200"

  auth_login_kubernetes {
    role = "opentofu-cicd"
    jwt  = file("/var/run/secrets/vault/token")
  }
}
```

## Conclusion

Vault Kubernetes auth provides zero-secret authentication for OpenTofu running in Kubernetes environments. The bound service account name and namespace constraints ensure only authorized workloads can authenticate. Combined with Vault's dynamic secrets engines, OpenTofu can obtain short-lived AWS credentials or database passwords without storing any long-lived secrets in the cluster.
