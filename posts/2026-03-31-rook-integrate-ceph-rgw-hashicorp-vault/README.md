# How to Integrate Ceph RGW with HashiCorp Vault

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Vault, Encryption, Security

Description: Learn how to integrate Ceph RGW with HashiCorp Vault for server-side encryption key management, enabling secure and auditable object encryption.

---

## Overview

HashiCorp Vault is a widely used secrets management platform. Integrating it with Ceph RGW enables Server-Side Encryption (SSE-KMS) where encryption keys are stored and managed in Vault rather than locally. This provides key rotation, access policies, and audit logging for all encryption key operations.

## Architecture

```
Client --> RGW (SSE-KMS) --> Vault (Transit secrets engine) --> Key returned --> Object encrypted
```

RGW uses Vault's Transit secrets engine to generate data encryption keys (DEKs) that encrypt each object.

## Setting Up Vault Transit Engine

```bash
# Enable the Transit secrets engine
vault secrets enable transit

# Create an encryption key for RGW
vault write -f transit/keys/rgw-encryption-key \
  type=aes256-gcm96

# Verify the key was created
vault read transit/keys/rgw-encryption-key
```

## Creating Vault Policy for RGW

```bash
cat > rgw-policy.hcl << 'EOF'
path "transit/keys/rgw-encryption-key" {
  capabilities = ["read"]
}

path "transit/datakey/plaintext/rgw-encryption-key" {
  capabilities = ["update"]
}

path "transit/decrypt/rgw-encryption-key" {
  capabilities = ["update"]
}

path "transit/encrypt/rgw-encryption-key" {
  capabilities = ["update"]
}
EOF

vault policy write rgw-policy rgw-policy.hcl
```

## Creating a Vault Token for RGW

```bash
# Create a token with the RGW policy
vault token create \
  -policy=rgw-policy \
  -display-name=ceph-rgw \
  -ttl=87600h \
  -renewable=true

# Save the token value for use in Ceph config
```

## Configuring RGW for Vault

```bash
# Enable Vault KMS backend
ceph config set client.rgw rgw_crypt_s3_kms_backend vault

# Configure Vault address and authentication
ceph config set client.rgw rgw_crypt_vault_addr "http://vault-host:8200"
ceph config set client.rgw rgw_crypt_vault_auth token
ceph config set client.rgw rgw_crypt_vault_token VAULT_TOKEN

# Configure the Vault path for key operations
ceph config set client.rgw rgw_crypt_vault_secret_engine transit
ceph config set client.rgw rgw_crypt_vault_prefix "/v1/transit"
```

## Using Vault AppRole Authentication

For production, use AppRole instead of a static token:

```bash
# Enable AppRole auth method
vault auth enable approle

# Create an AppRole for RGW
vault write auth/approle/role/ceph-rgw \
  token_policies=rgw-policy \
  token_ttl=1h \
  token_max_ttl=24h

# Get role_id and secret_id
vault read auth/approle/role/ceph-rgw/role-id
vault write -f auth/approle/role/ceph-rgw/secret-id
```

Configure RGW to use AppRole:

```bash
ceph config set client.rgw rgw_crypt_vault_auth agent
ceph config set client.rgw rgw_crypt_vault_role_id "ROLE_ID"
ceph config set client.rgw rgw_crypt_vault_secret_id "SECRET_ID"
```

## Uploading Encrypted Objects

```bash
# Upload with SSE-KMS pointing to the Vault key
aws --endpoint-url http://rgw-host:80 s3 cp \
  sensitive.txt s3://mybucket/sensitive.txt \
  --sse aws:kms \
  --sse-kms-key-id "rgw-encryption-key"
```

## Verifying Vault Audit Logs

```bash
# Enable Vault audit logging
vault audit enable file file_path=/var/log/vault-audit.log

# View RGW key access in audit log
grep "rgw-encryption-key" /var/log/vault-audit.log | jq .
```

## Summary

Integrating Ceph RGW with HashiCorp Vault's Transit secrets engine provides secure, auditable server-side encryption. Configure RGW with the Vault address and authentication method, create policies that restrict RGW to only its encryption keys, and use AppRole authentication for production environments. All key operations are logged in Vault's audit trail, providing full visibility into encryption activity.
