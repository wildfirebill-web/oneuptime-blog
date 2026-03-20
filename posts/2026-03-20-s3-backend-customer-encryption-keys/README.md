# How to Configure S3 Backend with Customer-Provided Encryption Keys in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, AWS, Security, Encryption

Description: Learn how to configure the OpenTofu S3 backend with customer-provided encryption keys (SSE-C) for full control over the encryption keys used to protect your state files.

## Introduction

SSE-C (Server-Side Encryption with Customer-Provided Keys) gives you complete control over the encryption keys used to protect your S3 objects. Unlike SSE-S3 or SSE-KMS, with SSE-C you manage the keys entirely — AWS handles the encryption and decryption but never stores your keys. This guide covers configuring SSE-C for the OpenTofu S3 backend.

## How SSE-C Works

With SSE-C:
1. You provide the encryption key in every S3 request
2. AWS uses your key to encrypt/decrypt the object
3. AWS does NOT store the key — it's used only during the request
4. If you lose the key, the data is permanently inaccessible

## Step 1: Generate an Encryption Key

SSE-C requires a 256-bit (32-byte) AES key, base64-encoded:

```bash
# Generate a random 32-byte key and base64-encode it
openssl rand -base64 32

# Example output: mUbp9NXK8GNKvSvOCiSSWnBFN+pHAqBIJXnWkPMcuiI=

# Store this key securely in your secrets manager
aws secretsmanager create-secret \
  --name "terraform-state-ssec-key" \
  --secret-string "mUbp9NXK8GNKvSvOCiSSWnBFN+pHAqBIJXnWkPMcuiI="
```

**Critical**: Store this key in a secure location (AWS Secrets Manager, HashiCorp Vault). If lost, all encrypted state is permanently inaccessible.

## Step 2: Configure the S3 Backend with SSE-C

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "prod/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true

    # Customer-provided encryption key (base64-encoded 32-byte key)
    sse_customer_algorithm = "AES256"
    sse_customer_key       = var.ssec_key

    # MD5 of the key for integrity verification (optional but recommended)
    # sse_customer_key_md5 = var.ssec_key_md5
  }
}
```

## Step 3: Manage the Key Variable

```hcl
variable "ssec_key" {
  type        = string
  description = "Base64-encoded 32-byte AES key for SSE-C"
  sensitive   = true
}
```

```bash
# Retrieve the key from Secrets Manager and set as variable
export TF_VAR_ssec_key=$(aws secretsmanager get-secret-value \
  --secret-id "terraform-state-ssec-key" \
  --query 'SecretString' \
  --output text)

# Or set directly (less secure)
export TF_VAR_ssec_key="mUbp9NXK8GNKvSvOCiSSWnBFN+pHAqBIJXnWkPMcuiI="
```

## Step 4: Initialize and Apply

```bash
tofu init

# Verify state is accessible with the key
tofu state list

# Apply — state written with SSE-C
tofu apply
```

## Key Rotation for SSE-C

SSE-C key rotation requires re-uploading all objects with the new key:

```bash
# Step 1: Generate a new key
NEW_KEY=$(openssl rand -base64 32)

# Step 2: List all state files
aws s3 ls s3://my-terraform-state/ --recursive | grep terraform.tfstate

# Step 3: Copy each object with new key (AWS performs the key swap)
aws s3 cp \
  s3://my-terraform-state/prod/terraform.tfstate \
  s3://my-terraform-state/prod/terraform.tfstate \
  --sse-c AES256 \
  --sse-c-key $(echo $OLD_KEY) \
  --sse-c-copy-source AES256 \
  --sse-c-copy-source-key $(echo $OLD_KEY)

# This is equivalent to using the new key:
aws s3 cp \
  s3://my-terraform-state/prod/terraform.tfstate \
  s3://my-terraform-state/prod/terraform.tfstate \
  --sse-c AES256 \
  --sse-c-key $NEW_KEY

# Step 4: Update your secrets manager with the new key
aws secretsmanager update-secret \
  --secret-id "terraform-state-ssec-key" \
  --secret-string "$NEW_KEY"
```

## Comparison with Other Encryption Options

| Feature | SSE-S3 | SSE-KMS | SSE-C |
|---------|--------|---------|-------|
| Key management | AWS | AWS KMS | You |
| CloudTrail audit | No | Yes | Partial |
| Key rotation | Automatic | Configurable | Manual |
| Cost | Free | KMS charges | Free |
| Key loss risk | None | None | Total data loss |

## When to Use SSE-C

Use SSE-C when:
- Compliance requires keys never stored in AWS
- You need complete key custody
- Your organization has key management infrastructure outside AWS

Avoid SSE-C when:
- You don't have robust key management processes
- Multiple team members need to access state
- Automated key rotation is needed (complex with SSE-C)

## Conclusion

SSE-C for the S3 backend provides maximum control over your encryption keys — but also maximum responsibility. The key is never stored by AWS, giving you full custody, but also full risk. Only use SSE-C when your compliance requirements mandate it and when you have robust key management processes in place. For most production environments, SSE-KMS provides the right balance of control, auditability, and operational safety.
