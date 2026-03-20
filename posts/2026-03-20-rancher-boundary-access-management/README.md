# How to Set Up Rancher with Boundary for Access Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, boundary, hashicorp, access-management, zero-trust, kubernetes

Description: A guide to integrating Rancher with HashiCorp Boundary for dynamic, identity-based access management to Kubernetes clusters and workloads.

## Overview

HashiCorp Boundary provides identity-based access management for infrastructure, replacing traditional VPN and bastion host approaches with a modern zero-trust model. Integrating Boundary with Rancher-managed Kubernetes clusters provides dynamic, audited access to cluster API servers, internal services, and pods without exposing them directly to the network. This guide covers the integration.

## Architecture

```
User/Operator
     |
  [OIDC/LDAP Authentication]
     |
  Boundary Controller (public)
     |
  Boundary Worker (in private network)
     |
  [Dynamic credentials via Vault]
     |
  Kubernetes API / Internal Services
  (Rancher-managed clusters)
```

## Prerequisites

- HashiCorp Boundary v0.14+ (HCP Boundary or self-hosted)
- HashiCorp Vault (for dynamic credentials)
- Rancher v2.7+ with managed clusters
- OIDC identity provider configured

## Step 1: Deploy Boundary Controller

```bash
# Install Boundary controller using Helm
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install boundary hashicorp/boundary \
  --namespace boundary \
  --create-namespace \
  --set controller.enabled=true \
  --set worker.enabled=true \
  --set global.enabled=true
```

### Boundary Configuration

```hcl
# boundary-controller.hcl
controller {
  name        = "rancher-boundary-controller"
  description = "Boundary controller for Rancher clusters"

  database {
    url = "postgresql://boundary:${DB_PASSWORD}@postgres:5432/boundary"
  }
}

listener "tcp" {
  address     = "0.0.0.0:9200"
  purpose     = "api"
  tls_disable = false
  tls_cert_file = "/tls/cert.pem"
  tls_key_file  = "/tls/key.pem"
}

listener "tcp" {
  address = "0.0.0.0:9201"
  purpose = "cluster"
}

kms "transit" {
  purpose    = "root"
  address    = "https://vault.example.com"
  token      = "${VAULT_TOKEN}"
  mount_path = "transit/"
  key_name   = "boundary-root"
}
```

## Step 2: Deploy Boundary Workers in Kubernetes

Workers run inside the private network and proxy connections to targets:

```yaml
# Boundary worker deployment in Kubernetes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: boundary-worker
  namespace: boundary
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: boundary-worker
          image: hashicorp/boundary:latest
          args:
            - server
            - -config=/boundary/config.hcl
          env:
            - name: BOUNDARY_CLUSTER_ID
              valueFrom:
                secretKeyRef:
                  name: boundary-worker-credentials
                  key: cluster-id
          volumeMounts:
            - name: config
              mountPath: /boundary
      volumes:
        - name: config
          configMap:
            name: boundary-worker-config
```

```hcl
# boundary-worker.hcl
worker {
  name        = "k8s-worker-01"
  description = "Boundary worker in Kubernetes cluster"
  address     = "boundary-worker.boundary.svc:9202"

  initial_upstreams = ["boundary-controller.boundary.svc:9201"]

  tags {
    type   = ["kubernetes"]
    region = ["us-east-1"]
  }
}
```

## Step 3: Configure Boundary Resources via Terraform

```hcl
# boundary-resources.tf

# Organization scope
resource "boundary_scope" "org" {
  name        = "Engineering Organization"
  description = "Engineering team scope"
  scope_id    = boundary_scope.global.id
  auto_create_admin_role   = true
  auto_create_default_role = true
}

# Project scope for Kubernetes access
resource "boundary_scope" "kubernetes" {
  name     = "Kubernetes Clusters"
  scope_id = boundary_scope.org.id
  auto_create_admin_role   = true
  auto_create_default_role = true
}

# Auth method - OIDC
resource "boundary_auth_method_oidc" "corporate" {
  name          = "Corporate SSO"
  scope_id      = boundary_scope.org.id
  issuer        = "https://login.microsoftonline.com/${var.tenant_id}/v2.0"
  client_id     = var.oidc_client_id
  client_secret = var.oidc_client_secret
  signing_algorithms = ["RS256"]
  api_url_prefix = "https://boundary.example.com"
  is_primary_for_scope = true
}

# Host catalog for Kubernetes API targets
resource "boundary_host_catalog_static" "kubernetes" {
  name     = "Kubernetes API Servers"
  scope_id = boundary_scope.kubernetes.id
}

# Host: Production cluster API server
resource "boundary_host_static" "prod_cluster" {
  name            = "prod-us-east-01"
  host_catalog_id = boundary_host_catalog_static.kubernetes.id
  address         = "10.0.1.100"   # Internal API server IP
}

# Host set
resource "boundary_host_set_static" "production" {
  name            = "Production Clusters"
  host_catalog_id = boundary_host_catalog_static.kubernetes.id
  host_ids        = [boundary_host_static.prod_cluster.id]
}

# Target: Kubernetes API access
resource "boundary_target" "kubernetes_api" {
  name         = "Production Kubernetes API"
  type         = "tcp"
  scope_id     = boundary_scope.kubernetes.id
  default_port = 6443

  host_source_ids = [boundary_host_set_static.production.id]

  # Require Vault dynamic credentials
  brokered_credential_source_ids = [boundary_credential_library_vault.k8s_token.id]
}
```

## Step 4: Dynamic Kubernetes Credentials via Vault

```hcl
# vault-k8s-credentials.hcl
# Configure Vault to issue short-lived Kubernetes ServiceAccount tokens

# Vault Kubernetes auth backend
resource "vault_kubernetes_secret_backend" "k8s" {
  path       = "kubernetes"
  kubernetes_host = "https://k8s-api.example.com:6443"
  kubernetes_ca_cert = file("k8s-ca.crt")
  service_account_jwt = data.kubernetes_secret.vault_sa.data["token"]
}

resource "vault_kubernetes_secret_backend_role" "developer" {
  backend    = vault_kubernetes_secret_backend.k8s.path
  name       = "k8s-developer"
  allowed_kubernetes_namespaces = ["development", "staging"]
  kubernetes_role_name = "developer"    # Binds to ClusterRole
  token_ttl  = "1h"
  token_max_ttl = "4h"
}
```

```hcl
# Boundary credential library using Vault
resource "boundary_credential_library_vault" "k8s_token" {
  name            = "K8s Developer Token"
  credential_store_id = boundary_credential_store_vault.main.id
  path            = "kubernetes/creds/k8s-developer"
  http_method     = "GET"
  credential_type = "kubernetes"
}
```

## Step 5: Connect to Kubernetes via Boundary

```bash
# Authenticate with corporate SSO
boundary authenticate oidc \
  -auth-method-id=amoidc_xxxxxxxx \
  -addr=https://boundary.example.com

# List available Kubernetes targets
boundary targets list -scope-id=p_xxxxxxxxx

# Connect to production Kubernetes API
# Boundary creates a local proxy with dynamic credentials
boundary connect kubernetes \
  -target-id=ttcp_xxxxxxxxx \
  -k8s-connect-port=6443

# This sets KUBECONFIG automatically
kubectl get nodes   # Uses Boundary proxy with short-lived credentials
```

## Step 6: Audit Access

```bash
# View Boundary session logs
boundary sessions list -scope-id=p_xxxxxxxxx

# View specific session details
boundary sessions read -id=s_xxxxxxxxx

# Export for compliance
boundary sessions list -scope-id=p_xxxxxxxxx -format=json \
  | jq '.items[] | {user: .user_id, target: .target_id, start: .created_time, end: .terminated_time}'
```

## Conclusion

Integrating Rancher with HashiCorp Boundary replaces traditional VPN access with a zero-trust, identity-driven model. Boundary provides dynamic, short-lived credentials via Vault integration, full session auditing, and fine-grained access control by team and target cluster. The combination of Rancher for Kubernetes management and Boundary for access management creates a strong security posture for enterprise environments requiring detailed access control and audit trails.
