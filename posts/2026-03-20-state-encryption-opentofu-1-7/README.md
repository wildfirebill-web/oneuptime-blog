# How to Use State Encryption Introduced in OpenTofu 1.7

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, State Encryption, Security, OpenTofu 1.7, Infrastructure as Code

Description: Learn how to enable and configure native state encryption introduced in OpenTofu 1.7 to protect sensitive values in your Terraform state files.

## Introduction

OpenTofu 1.7 introduced native state encryption as a first-class feature. State files often contain sensitive values like database passwords, API keys, and private keys. Encrypting the state at rest adds a critical security layer independent of your backend's encryption.

## Basic State Encryption with PBKDF2

The simplest encryption uses a passphrase with PBKDF2 key derivation.

```hcl
# main.tf
terraform {
  encryption {
    key_provider "pbkdf2" "main" {
      passphrase = var.state_encryption_passphrase

      # Key derivation parameters
      key_length   = 32  # 256-bit key
      iterations   = 600000
      salt_length  = 32
      hash_function = "sha512"
    }

    method "aes_gcm" "main" {
      keys = key_provider.pbkdf2.main
    }

    state {
      method = method.aes_gcm.main

      # Allow reading unencrypted state during migration
      enforced = false  # set to true after all state is encrypted
    }
  }
}

variable "state_encryption_passphrase" {
  type      = string
  sensitive = true
}
```

## Using AWS KMS for Key Management

```hcl
terraform {
  encryption {
    key_provider "aws_kms" "main" {
      kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
      region     = "us-east-1"

      # Key context for additional access control
      key_spec = "AES_256"
    }

    method "aes_gcm" "main" {
      keys = key_provider.aws_kms.main
    }

    state {
      method   = method.aes_gcm.main
      enforced = true
    }

    # Also encrypt plan files
    plan {
      method   = method.aes_gcm.main
      enforced = true
    }
  }
}
```

## Using GCP KMS

```hcl
terraform {
  encryption {
    key_provider "gcp_kms" "main" {
      kms_encryption_key = "projects/my-project/locations/us-east1/keyRings/my-ring/cryptoKeys/my-key"
    }

    method "aes_gcm" "main" {
      keys = key_provider.gcp_kms.main
    }

    state {
      method   = method.aes_gcm.main
      enforced = true
    }
  }
}
```

## Key Rotation

Rotate encryption keys without decrypting and re-encrypting manually.

```hcl
terraform {
  encryption {
    key_provider "aws_kms" "new_key" {
      kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/new-key-id"
      region     = "us-east-1"
    }

    key_provider "aws_kms" "old_key" {
      kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/old-key-id"
      region     = "us-east-1"
    }

    method "aes_gcm" "main" {
      # Use new key for writing; accept old key for reading during rotation
      keys = key_provider.aws_kms.new_key
    }

    method "aes_gcm" "fallback" {
      keys = key_provider.aws_kms.old_key
    }

    state {
      method = method.aes_gcm.main

      # Read state encrypted with old key
      fallback {
        method = method.aes_gcm.fallback
      }
    }
  }
}
```

## Migrating Existing Unencrypted State

```bash
# Step 1: Add encryption config with enforced = false
# Step 2: Run tofu apply to re-encrypt state
tofu apply -refresh=false

# Step 3: Set enforced = true to prevent reading unencrypted state
# Step 4: Commit the updated configuration
```

## Summary

OpenTofu 1.7 native state encryption protects sensitive values in state files using PBKDF2 passphrases or cloud KMS keys. The `enforced` flag controls migration safety, and key fallbacks enable smooth key rotation. This feature adds encryption independent of backend storage, protecting state even if your S3 bucket or storage backend is compromised.
