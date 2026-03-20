# How to Configure Kubernetes Secrets with External Vaults in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Secrets, HashiCorp Vault, External Secrets, Security

Description: Configure Kubernetes secrets management using external vaults in Rancher, integrating HashiCorp Vault and the External Secrets Operator for centralized, auditable secret storage.

## Introduction

Kubernetes native Secrets are base64-encoded, not encrypted at rest by default, and lack rotation, auditing, and access control capabilities that production security requires. External vault solutions like HashiCorp Vault provide encryption, dynamic secrets, audit logging, and fine-grained access policies. This guide covers integrating external vaults with Kubernetes clusters managed by Rancher.

## Architecture Overview

```
┌─────────────────────────────────────────┐
│  Kubernetes Cluster (Rancher-managed)   │
│                                         │
│  ┌─────────────┐    ┌────────────────┐  │
│  │ Application │───▶│  K8s Secret    │  │
│  │    Pod      │    │  (synced copy) │  │
│  └─────────────┘    └───────┬────────┘  │
│                             │ sync      │
│  ┌─────────────────────────▼──────────┐ │
│  │   External Secrets Operator (ESO)  │ │
│  └─────────────────────────┬──────────┘ │
└─────────────────────────────┼──────────┘
                              │ fetch
                    ┌─────────▼─────────┐
                    │  HashiCorp Vault   │
                    │  (external)        │
                    └───────────────────┘
```

## Option 1: External Secrets Operator with HashiCorp Vault

### Install External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true
```

### Configure Vault Authentication

```bash
# Enable Kubernetes auth in Vault
vault auth enable kubernetes

# Configure the Kubernetes auth method
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token

# Create a Vault policy for secret access
vault policy write app-secrets - <<EOF
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
EOF

# Bind the policy to the Kubernetes service account
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=production \
  policies=app-secrets \
  ttl=1h
```

### Create SecretStore

```yaml
# secretstore.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.company.com:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp"
          serviceAccountRef:
            name: "myapp-sa"
```

### Create ExternalSecret

```yaml
# externalsecret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: myapp-secret
    creationPolicy: Owner
  data:
    - secretKey: database-password
      remoteRef:
        key: secret/data/myapp/database
        property: password
    - secretKey: api-key
      remoteRef:
        key: secret/data/myapp/api
        property: key
```

## Option 2: Vault Agent Injector

The Vault Agent Injector uses a mutating webhook to inject secrets directly into pod filesystems:

```yaml
# pod with vault annotations
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: production
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-config: "secret/data/myapp/database"
    vault.hashicorp.com/agent-inject-template-config: |
      {{- with secret "secret/data/myapp/database" -}}
      export DB_PASSWORD="{{ .Data.data.password }}"
      export DB_HOST="{{ .Data.data.host }}"
      {{- end }}
spec:
  serviceAccountName: myapp-sa
  containers:
    - name: app
      image: myregistry/myapp:latest
      command: ["/bin/sh", "-c", "source /vault/secrets/config && ./start.sh"]
```

## Option 3: AWS Secrets Manager via ESO

For clusters running on AWS:

```yaml
# ClusterSecretStore for AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

## Step 4: Enable Secret Rotation

Configure automatic rotation by setting a short refresh interval:

```yaml
spec:
  refreshInterval: 15m    # Sync every 15 minutes from Vault
```

Force immediate sync:

```bash
# Trigger immediate secret refresh
kubectl annotate externalsecret myapp-secrets \
  force-sync=$(date +%s) \
  --overwrite \
  -n production
```

## Step 5: Monitor Secret Sync Status

```bash
# Check ExternalSecret sync status
kubectl get externalsecrets -n production

# View sync details
kubectl describe externalsecret myapp-secrets -n production

# Check ESO metrics
kubectl port-forward svc/external-secrets-metrics 8080:8080 -n external-secrets
```

## Conclusion

External vaults combined with the External Secrets Operator provide enterprise-grade secret management for Rancher-managed Kubernetes clusters. Secrets are centrally managed in Vault with full audit trails, access policies, and automatic rotation, while Kubernetes applications consume them transparently as native Secrets. This approach satisfies compliance requirements for PCI-DSS, SOC 2, and HIPAA.
