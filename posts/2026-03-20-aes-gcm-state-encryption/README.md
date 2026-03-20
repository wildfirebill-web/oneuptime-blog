# How to Use AES-GCM Encryption Method for State in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Security, Encryption

Description: Learn how AES-GCM works as the encryption method in OpenTofu state encryption and how to configure it with different key providers for authenticated encryption.

## Introduction

AES-GCM (Advanced Encryption Standard with Galois/Counter Mode) is the encryption algorithm used by OpenTofu for state and plan file encryption. It provides both confidentiality and data integrity (authenticated encryption), making it resistant to tampering. This guide explains the method configuration and its options.

## Why AES-GCM?

AES-GCM provides:
- **Confidentiality**: Data cannot be read without the key
- **Integrity**: Any modification to the ciphertext is detectable
- **Authentication**: Verifies the data came from someone with the key
- **Performance**: Hardware acceleration (AES-NI) on modern CPUs

## Basic AES-GCM Configuration

The `aes_gcm` method block appears between the key provider and the state/plan directives:

```hcl
terraform {
  encryption {
    # Key provider (provides the encryption key)
    key_provider "pbkdf2" "my_key" {
      passphrase = var.passphrase
    }

    # AES-GCM method (uses the key to encrypt/decrypt)
    method "aes_gcm" "my_method" {
      keys = key_provider.pbkdf2.my_key

      # Optional: additional authenticated data (AAD) for extra protection
      # aad = "my-unique-aad-string"
    }

    # Apply to state files
    state {
      method  = method.aes_gcm.my_method
      enforced = true  # Fail if encryption cannot be applied
    }

    # Apply to plan files
    plan {
      method  = method.aes_gcm.my_method
      enforced = true
    }
  }
}
```

## Understanding Key Size

AES-GCM with different key providers uses different key sizes:

```hcl
# PBKDF2 with 256-bit key (default)
key_provider "pbkdf2" "key_256" {
  passphrase = var.passphrase
  key_length = 32  # 32 bytes = 256 bits
}

# AWS KMS data keys are 256-bit by default
key_provider "aws_kms" "key" {
  kms_key_id = "alias/my-key"
  region     = "us-east-1"
  key_spec   = "AES_256"  # Explicitly specify 256-bit
}
```

## Using Multiple Methods for Migration

During key rotation, you can configure fallback methods:

```hcl
terraform {
  encryption {
    # New key provider
    key_provider "pbkdf2" "new_key" {
      passphrase = var.new_passphrase
    }

    # Old key provider (for reading old state)
    key_provider "pbkdf2" "old_key" {
      passphrase = var.old_passphrase
    }

    # New method
    method "aes_gcm" "new_method" {
      keys = key_provider.pbkdf2.new_key
    }

    # Old method (for decryption fallback)
    method "aes_gcm" "old_method" {
      keys = key_provider.pbkdf2.old_key
    }

    state {
      method = method.aes_gcm.new_method

      # Fallback to old method for reading state encrypted with old key
      fallback {
        method = method.aes_gcm.old_method
      }
    }
  }
}
```

## Enforced vs Non-Enforced Encryption

The `enforced` parameter controls behavior when encryption fails:

```hcl
state {
  method   = method.aes_gcm.my_method
  enforced = true   # Default: true — fail if encryption cannot be applied
}

# With enforced = false, OpenTofu will read unencrypted state
# Useful during migration from unencrypted to encrypted state
state {
  method   = method.aes_gcm.my_method
  enforced = false  # Allow reading unencrypted state (migration mode)
}
```

## Understanding the Encryption Process

When OpenTofu writes state:

1. Key provider generates or retrieves a data key
2. AES-GCM encrypts the state JSON with the data key
3. The encrypted state and key metadata are stored together
4. A nonce (random) is generated for each encryption operation

When OpenTofu reads state:

1. The key provider metadata is extracted
2. The key provider decrypts or retrieves the data key
3. AES-GCM decrypts and verifies the state
4. OpenTofu parses the decrypted JSON

## Verifying the Encrypted State Format

```bash
# View the raw encrypted state (binary/base64)
cat terraform.tfstate | head -5

# Verify OpenTofu can still read it
tofu state list
tofu show
```

## Performance Considerations

AES-GCM with hardware acceleration is very fast:
- A 1MB state file encrypts in < 1ms on modern hardware
- No meaningful performance impact on OpenTofu operations
- Key derivation (PBKDF2) takes 100-500ms per operation, not per byte

## Conclusion

AES-GCM is an excellent choice for state file encryption, providing both confidentiality and integrity guarantees. The OpenTofu encryption framework makes it straightforward to combine AES-GCM with any supported key provider. Use `enforced = true` in production to ensure state is never stored unencrypted, and configure fallback methods during key rotation to maintain smooth operations.
