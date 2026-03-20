# How to Manage Secrets Safely with Ephemeral Resources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Ephemeral Resources, Secrets Management, Security, Infrastructure as Code, DevOps

Description: A guide to best practices for managing secrets safely in OpenTofu using ephemeral resources to prevent sensitive data from appearing in state files.

## Introduction

Managing secrets in infrastructure code is one of the most critical security challenges. OpenTofu's ephemeral resources provide a way to use secrets during deployments without persisting them to state files. This guide covers patterns for safely handling passwords, API keys, certificates, and other sensitive values.

## The Problem with Regular Data Sources

```hcl
# INSECURE: Secret value is stored in state file

data "aws_secretsmanager_secret_version" "db_pass" {
  secret_id = "myapp/db-password"
}

resource "aws_db_instance" "main" {
  password = jsondecode(data.aws_secretsmanager_secret_version.db_pass.secret_string).password
  # Password stored in: terraform.tfstate and remote state backend
}

# Anyone with read access to state can see the password!
```

## The Ephemeral Solution

```hcl
# SECURE: Secret value never enters state file
ephemeral "aws_secretsmanager_secret_version" "db_pass" {
  secret_id = "myapp/db-password"
}

resource "aws_db_instance" "main" {
  password = jsondecode(
    ephemeral.aws_secretsmanager_secret_version.db_pass.secret_string
  ).password

  lifecycle {
    # Don't track password changes in state
    ignore_changes = [password]
  }
}
```

## Pattern 1: Database Credentials

```hcl
ephemeral "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = "${var.environment}/database/credentials"
}

locals {
  db_creds = jsondecode(
    ephemeral.aws_secretsmanager_secret_version.db_creds.secret_string
  )
}

resource "aws_db_instance" "main" {
  identifier        = "myapp-${var.environment}"
  engine            = "postgres"
  engine_version    = "15"
  instance_class    = var.db_instance_class
  allocated_storage = var.db_storage
  db_name           = var.db_name
  username          = local.db_creds.username
  password          = local.db_creds.password  # Not in state

  lifecycle {
    ignore_changes = [username, password]
  }
}
```

## Pattern 2: API Keys for External Services

```hcl
ephemeral "vault_kv_secret_v2" "api_keys" {
  mount = "secret"
  name  = "myapp/${var.environment}/api-keys"
}

locals {
  keys = jsondecode(ephemeral.vault_kv_secret_v2.api_keys.data_json)
}

# Configure external providers with ephemeral keys
provider "datadog" {
  api_key = local.keys.datadog_api_key
  app_key = local.keys.datadog_app_key
}

resource "aws_ssm_parameter" "stripe_key" {
  name  = "/myapp/${var.environment}/stripe-api-key"
  type  = "SecureString"
  value = local.keys.stripe_api_key  # Stored encrypted in SSM
  # The original plaintext is ephemeral
}
```

## Pattern 3: TLS Certificates

```hcl
# Generate certificate private key ephemerally
ephemeral "tls_private_key" "app_cert" {
  algorithm   = "ECDSA"
  ecdsa_curve = "P256"
}

resource "tls_self_signed_cert" "app" {
  private_key_pem = ephemeral.tls_private_key.app_cert.private_key_pem

  subject {
    common_name  = "app.${var.domain}"
    organization = var.organization
  }

  validity_period_hours = 8760  # 1 year

  allowed_uses = [
    "key_encipherment",
    "digital_signature",
    "server_auth",
  ]
}

# Store private key securely - NOT in state
resource "aws_secretsmanager_secret_version" "app_cert_key" {
  secret_id = aws_secretsmanager_secret.app_cert_key.id
  secret_string = jsonencode({
    private_key = ephemeral.tls_private_key.app_cert.private_key_pem
  })
}
```

## Pattern 4: Secrets Rotation

```hcl
# Always fetch the current version - works with rotation
data "aws_secretsmanager_secret" "app" {
  name = "myapp/${var.environment}/config"
}

ephemeral "aws_secretsmanager_secret_version" "app" {
  secret_id = data.aws_secretsmanager_secret.app.id
  # No version_id = always gets the latest (rotated) version
}

# Each apply uses the current secret value
# If secrets manager rotates it between applies, new value is used
resource "aws_ecs_task_definition" "app" {
  family = "myapp"

  container_definitions = jsonencode([{
    name = "app"
    environment = [{
      name  = "CONFIG"
      value = ephemeral.aws_secretsmanager_secret_version.app.secret_string
    }]
  }])
}
```

## Pattern 5: Vault Dynamic Secrets

```hcl
# Vault generates unique, time-limited credentials
ephemeral "vault_database_secret" "app" {
  mount = "database"
  name  = "myapp-role"
  # Vault creates a new credential each time
  # Automatically expires after lease duration
}

resource "aws_db_instance" "app_replica" {
  # Use dynamically generated credentials
  username = ephemeral.vault_database_secret.app.username
  password = ephemeral.vault_database_secret.app.password

  lifecycle {
    ignore_changes = [username, password]
  }
}
```

## Auditing Secret Access

```bash
# Ephemeral resources still generate API calls to secret stores
# These are auditable in:

# AWS CloudTrail for Secrets Manager access:
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetSecretValue

# Vault audit logs:
vault audit list
cat /var/log/vault/audit.log | jq '.request.path'
```

## State File Security Checklist

```bash
# After migration to ephemeral resources:

# 1. Rotate all secrets that were previously in state
aws secretsmanager rotate-secret --secret-id myapp/db-password

# 2. Verify new state doesn't contain sensitive data
tofu state pull | grep -i "password\|secret\|key\|token"
# Should return no matches

# 3. Enable state encryption (OpenTofu 1.7+)
# In tofu configuration:
# terraform {
#   encryption {
#     key_provider "pbkdf2" "main" {
#       passphrase = var.state_passphrase
#     }
#     method "aes_gcm" "main" {
#       keys = key_provider.pbkdf2.main
#     }
#     state {
#       method = method.aes_gcm.main
#     }
#   }
# }
```

## Conclusion

Ephemeral resources represent a paradigm shift in secrets management for infrastructure as code. By using ephemeral resources for all sensitive values, you eliminate the risk of secrets appearing in state files while maintaining the declarative, idempotent nature of OpenTofu. Combine ephemeral resources with write-only attributes (for resources that support them), secret rotation, and state encryption for a comprehensive secrets management strategy. The small additional complexity of using ephemeral resources is well worth the significant security improvement they provide.
