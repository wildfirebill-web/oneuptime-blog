# How to Set Up Server-Side Encryption with KMS (SSE-KMS) for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, SSE-KMS, KMS, Encryption, HashiCorp Vault

Description: Learn how to configure SSE-KMS server-side encryption for Ceph RADOS Gateway using HashiCorp Vault as the key management service for centralized key control.

---

## What is SSE-KMS?

SSE-KMS (Server-Side Encryption with KMS-Managed Keys) is an S3-compatible encryption model where a dedicated Key Management Service (KMS) manages encryption keys. Unlike SSE-S3, customers can audit key usage, control access policies, and rotate keys via the KMS. Ceph RGW supports SSE-KMS through integration with HashiCorp Vault and other KMIP-compatible systems.

## Architecture

```
Client --> Ceph RGW --> HashiCorp Vault (key retrieval)
                   --> OSD (encrypted data storage)
```

When an object is uploaded with SSE-KMS, RGW fetches the specified key from Vault, encrypts the object, and stores only encrypted data on the OSD.

## Step 1: Configure Vault

Enable the Transit secrets engine:

```bash
vault secrets enable transit
vault write -f transit/keys/rgw-master-key type=aes256-gcm96
```

Create a policy for RGW:

```bash
vault policy write rgw-policy - <<EOF
path "transit/keys/rgw-master-key" {
  capabilities = ["read"]
}
path "transit/encrypt/rgw-master-key" {
  capabilities = ["update"]
}
path "transit/decrypt/rgw-master-key" {
  capabilities = ["update"]
}
EOF
```

## Step 2: Configure Ceph RGW for Vault

```bash
ceph config set client.rgw rgw_crypt_vault_addr "https://vault.example.com:8200"
ceph config set client.rgw rgw_crypt_vault_auth "token"
ceph config set client.rgw rgw_crypt_vault_token_file "/etc/ceph/vault.token"
ceph config set client.rgw rgw_crypt_vault_prefix "/v1/transit"
ceph config set client.rgw rgw_crypt_sse_kms_backend "vault"
```

Store the Vault token:

```bash
echo "hvs.your-vault-token" > /etc/ceph/vault.token
chmod 600 /etc/ceph/vault.token
```

## Step 3: Restart RGW

```bash
ceph orch restart rgw.default
```

## Step 4: Upload with SSE-KMS

```bash
aws s3 cp myfile.txt s3://mybucket/myfile.txt \
  --sse aws:kms \
  --sse-kms-key-id "rgw-master-key" \
  --endpoint-url https://rgw.example.com
```

## Step 5: Verify Encryption

```bash
aws s3api head-object \
  --bucket mybucket \
  --key myfile.txt \
  --endpoint-url https://rgw.example.com
```

Expected response:
```json
{
  "ServerSideEncryption": "aws:kms",
  "SSEKMSKeyId": "rgw-master-key"
}
```

## Key Rotation with Vault

Rotate the master key in Vault without re-encrypting objects:

```bash
vault write -f transit/keys/rgw-master-key/rotate
```

Vault keeps all previous key versions for decryption, so existing objects remain accessible.

## Summary

SSE-KMS for Ceph RGW uses HashiCorp Vault's Transit secrets engine to manage encryption keys centrally. Configure Vault with the Transit engine and a policy, set Ceph RGW config to point at Vault, then upload objects using `--sse aws:kms`. Key rotation is handled by Vault without requiring object re-encryption, providing strong auditability and centralized control over data encryption keys.
