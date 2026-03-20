# How to Avoid Storing Secrets in OpenTofu State

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Security, Secrets Management, State, Infrastructure as Code

Description: Learn how to prevent secrets from being stored in the OpenTofu state file using ephemeral resources, write-only attributes, and external secret stores.

## Introduction

The OpenTofu state file contains the full attribute values of every managed resource. When a resource has a password, API key, or other secret as an attribute, that secret ends up in the state file in plaintext (or only backend-encrypted). This creates a significant security risk if state files are accessed by unauthorized users or committed to version control.

## The Problem: Secrets in State

Understanding what ends up in state.

```hcl
# This stores the password in state as a plaintext value
resource "aws_db_instance" "main" {
  identifier = "myapp-db"
  password   = var.db_password  # ends up in terraform.tfstate as plaintext
}

# terraform.tfstate (contains the secret)
# {
#   "resources": [{
#     "type": "aws_db_instance",
#     "instances": [{
#       "attributes": {
#         "password": "my-super-secret-password"  ← in state
#       }
#     }]
#   }]
# }
```

## Solution 1: Use Write-Only Attributes (OpenTofu 1.11+)

Write-only attributes are sent to the provider but never stored in state.

```hcl
resource "aws_db_instance" "main" {
  identifier = "myapp-db"
  engine     = "postgres"
  username   = "admin"

  # Use write-only attribute - never stored in state
  password_wo         = var.db_password
  password_wo_version = var.db_password_version
}
```

## Solution 2: Use Ephemeral Resources (OpenTofu 1.11+)

Retrieve secrets from a secret store without touching state.

```hcl
# Fetch password from Secrets Manager - never stored in state
ephemeral "aws_secretsmanager_secret_version" "db_pass" {
  secret_id = "prod/myapp/db-password"
}

resource "aws_db_instance" "main" {
  identifier = "myapp-db"
  engine     = "postgres"
  username   = "admin"

  # Combine ephemeral source + write-only attribute
  password_wo         = ephemeral.aws_secretsmanager_secret_version.db_pass.secret_string
  password_wo_version = var.db_password_version
}
```

## Solution 3: External Secret Store Integration

Create secrets in a secret manager separately from the database.

```hcl
# Create the secret first
resource "aws_secretsmanager_secret" "db_password" {
  name        = "prod/myapp/db-password"
  description = "RDS master password"
}

# Store a random password in the secret
resource "random_password" "db" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db.result
}

# NOTE: random_password.db.result IS stored in state
# For higher security, generate passwords externally and inject via
# write-only attributes or ephemeral resources
```

## Solution 4: Encrypt Your State Backend

If secrets do end up in state, ensure the backend is encrypted.

```hcl
terraform {
  backend "s3" {
    bucket     = "my-tofu-state"
    key        = "prod/terraform.tfstate"
    region     = "us-east-1"
    encrypt    = true           # server-side encryption
    kms_key_id = "arn:aws:kms:..." # use a dedicated KMS key
  }

  # Application-level state encryption (OpenTofu 1.7+)
  encryption {
    key_provider "aws_kms" "main" {
      kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/my-key"
    }

    method "aes_gcm" "default" {
      keys = key_provider.aws_kms.main
    }

    state {
      method = method.aes_gcm.default
    }
  }
}
```

## Auditing Secrets in Existing State

Check if secrets are currently in your state file.

```bash
# Pull and search for potential secret attributes
tofu state pull | jq '.resources[].instances[].attributes | keys[]' | \
  grep -i "password\|secret\|key\|token"

# If found, plan to migrate to write-only attributes or ephemeral resources
# and manually remove from state using tofu state rm + reimport
```

## Summary

Preventing secrets from reaching the state file requires using write-only attributes (`password_wo`) for resources that support them, fetching secrets from external stores via ephemeral resources, and storing randomly generated passwords in dedicated secret stores rather than exposing them through state. As a fallback, encrypt your state backend with KMS and enable OpenTofu's state encryption feature. Audit existing state files periodically to identify secrets that need to be migrated.
