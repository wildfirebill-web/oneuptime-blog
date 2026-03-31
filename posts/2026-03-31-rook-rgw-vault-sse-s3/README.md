# How to Configure Vault Integration for SSE-S3 in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Vault, Encryption, Security

Description: Configure HashiCorp Vault as the KMS backend for SSE-S3 encryption in Ceph RGW, including token, AppRole, and Kubernetes authentication methods.

---

HashiCorp Vault is the most common KMS backend for Ceph RGW SSE-S3 encryption. This guide covers configuring Vault integration with multiple authentication methods.

## Vault Authentication Methods

RGW supports three Vault auth methods:
- **token** - Static token (simple but requires rotation)
- **agent** - Vault Agent sidecar manages token renewal
- **kubernetes** - Kubernetes service account authentication (recommended for Rook)

## Token-Based Vault Authentication

```bash
# Enable Vault KMS backend
ceph config set client.rgw rgw_crypt_s3_kms_backend vault

# Vault server address
ceph config set client.rgw rgw_crypt_vault_addr https://vault.example.com:8200

# Authentication method
ceph config set client.rgw rgw_crypt_vault_auth token

# Vault token (store securely)
ceph config set client.rgw rgw_crypt_vault_token hvs.VAULT_TOKEN_HERE

# Vault KV path for encryption keys
ceph config set client.rgw rgw_crypt_vault_prefix /v1/secret/data/ceph
ceph config set client.rgw rgw_crypt_vault_secret_engine kv
```

## Kubernetes-Based Vault Authentication

For Rook deployments, Kubernetes auth is preferred:

```bash
# Enable Vault Kubernetes auth
ceph config set client.rgw rgw_crypt_vault_auth kubernetes

# Vault role for the RGW service account
ceph config set client.rgw rgw_crypt_vault_ssl_cacert /etc/ceph/vault-ca.crt
```

Configure Vault to accept the RGW Kubernetes service account:

```bash
# Enable Kubernetes auth in Vault
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create a policy for RGW
vault policy write ceph-rgw-policy - <<EOF
path "secret/data/ceph/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
EOF

# Create a role binding the RGW service account
vault write auth/kubernetes/role/ceph-rgw \
  bound_service_account_names=rook-ceph-rgw \
  bound_service_account_namespaces=rook-ceph \
  policies=ceph-rgw-policy \
  ttl=1h
```

## Creating Vault Secrets for Encryption

```bash
# Enable the KV secrets engine
vault secrets enable -path=secret kv-v2

# Create an encryption key for RGW
vault kv put secret/ceph/my-key \
  key=$(openssl rand -base64 32)
```

## Applying in Rook

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [client.rgw.my-store.a]
    rgw_crypt_s3_kms_backend = vault
    rgw_crypt_vault_addr = https://vault.example.com:8200
    rgw_crypt_vault_auth = kubernetes
    rgw_crypt_vault_prefix = /v1/secret/data/ceph
    rgw_crypt_vault_secret_engine = kv
```

## Testing Vault Encryption

```bash
# Upload an object with SSE-S3 using Vault key
aws s3 cp testfile.txt s3://secure-bucket/testfile.txt \
  --sse AES256 \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc

# Verify encryption
aws s3api head-object --bucket secure-bucket --key testfile.txt \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc \
  --query 'ServerSideEncryption'
```

## Summary

HashiCorp Vault integrates with Ceph RGW via `rgw_crypt_s3_kms_backend = vault`. For Rook deployments, use Kubernetes-based Vault authentication to avoid managing static tokens. Configure the Vault role to bind the RGW service account and set a Vault policy with least-privilege access to encryption key paths.
