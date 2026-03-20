# How to Configure State Encryption with OpenBao in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Security, Encryption, OpenBao

Description: Learn how to configure OpenTofu state file encryption using OpenBao (the open-source HashiCorp Vault fork) for self-hosted secret and key management.

## Introduction

OpenBao is an open-source fork of HashiCorp Vault that provides secret management and encryption services. When you want full control over your key management infrastructure without relying on cloud KMS services, OpenBao is an excellent choice. OpenTofu supports OpenBao as a key provider for state encryption.

## Prerequisites

- OpenBao server running and unsealed
- OpenBao Transit secrets engine enabled
- Authentication method configured (Token, AppRole, etc.)
- OpenTofu 1.7+

## Step 1: Set Up OpenBao Transit Secrets Engine

```bash
# Enable the Transit secrets engine in OpenBao

bao secrets enable transit

# Create a named encryption key for OpenTofu state
bao write transit/keys/terraform-state \
  type=aes256-gcm96 \
  auto_rotate_period=2592000  # Auto-rotate every 30 days

# Verify the key was created
bao read transit/keys/terraform-state
```

## Step 2: Create an OpenBao Policy

```bash
# Create a policy file
cat > terraform-state-policy.hcl << 'EOF'
# Allow OpenTofu to encrypt/decrypt with the key
path "transit/encrypt/terraform-state" {
  capabilities = ["create", "update"]
}

path "transit/decrypt/terraform-state" {
  capabilities = ["create", "update"]
}

path "transit/keys/terraform-state" {
  capabilities = ["read"]
}
EOF

# Apply the policy
bao policy write terraform-state terraform-state-policy.hcl
```

## Step 3: Configure Authentication for OpenTofu

### Using AppRole Authentication

```bash
# Enable AppRole auth method
bao auth enable approle

# Create an AppRole for OpenTofu
bao write auth/approle/role/opentofu \
  policies=terraform-state \
  token_ttl=1h \
  token_max_ttl=4h

# Get Role ID and Secret ID
bao read auth/approle/role/opentofu/role-id
bao write -f auth/approle/role/opentofu/secret-id
```

### Using Token Authentication

```bash
# Create a token with the terraform-state policy
bao token create \
  -policy=terraform-state \
  -period=24h \
  -orphan
```

## Step 4: Configure the OpenBao Key Provider

```hcl
# encryption.tf
terraform {
  required_version = ">= 1.7.0"

  encryption {
    key_provider "openbao" "state_key" {
      # OpenBao server address
      address = "https://openbao.example.com:8200"

      # Transit key name
      transit_key_name = "terraform-state"

      # Transit secrets engine mount path (default: "transit")
      # transit_mount_path = "transit"

      # Authentication - using token
      token = var.openbao_token

      # Or using AppRole
      # auth_login_path = "auth/approle/login"
      # auth_login_params = {
      #   role_id   = var.openbao_role_id
      #   secret_id = var.openbao_secret_id
      # }

      # Skip TLS verification (for self-signed certs in dev)
      # skip_tls_verify = true

      # Custom CA certificate
      # ca_cert_file = "/path/to/ca.crt"
    }

    method "aes_gcm" "state_method" {
      keys = key_provider.openbao.state_key
    }

    state {
      method = method.aes_gcm.state_method
    }

    plan {
      method = method.aes_gcm.state_method
    }
  }
}
```

## Step 5: Define Variables for Secrets

```hcl
variable "openbao_token" {
  type      = string
  sensitive = true
}

# Or for AppRole
variable "openbao_role_id" {
  type      = string
  sensitive = true
}

variable "openbao_secret_id" {
  type      = string
  sensitive = true
}
```

```bash
# Set environment variables
export TF_VAR_openbao_token="s.XXXXX"
export OPENBAO_ADDR="https://openbao.example.com:8200"
export OPENBAO_TOKEN="s.XXXXX"
```

## Step 6: Apply and Verify

```bash
# Initialize
tofu init

# Plan and apply
tofu plan
tofu apply

# Verify encryption by checking OpenBao audit logs
bao audit list
```

## High Availability Setup

For production, run OpenBao in HA mode:

```hcl
key_provider "openbao" "state_key" {
  address          = "https://openbao-lb.example.com:8200"
  transit_key_name = "terraform-state"
  token            = var.openbao_token

  # Retry configuration
  max_retries = 3
  retry_wait_min = 0
  retry_wait_max = 30
}
```

## Key Rotation

OpenBao Transit automatically manages key versions. To manually rotate:

```bash
# Rotate the encryption key
bao write -f transit/keys/terraform-state/rotate

# Rewrap all existing encrypted data with the new key
bao write transit/rewrap/terraform-state \
  ciphertext="<existing_ciphertext>"
```

## Conclusion

OpenBao provides a self-hosted, open-source alternative to cloud KMS services for OpenTofu state encryption. It's ideal for organizations that need full control over their key management infrastructure or operate in air-gapped environments. The Transit secrets engine provides strong encryption, automatic key rotation, and detailed audit logs - all the features needed for production-grade state encryption.
