# How to Understand Ephemeral Resources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Ephemeral Resources, State, Security, Infrastructure as Code, DevOps

Description: A guide to understanding ephemeral resources in OpenTofu, how they differ from regular resources and data sources, and when to use them.

## Introduction

Ephemeral resources are a special type of resource introduced in OpenTofu that exist only for the duration of a plan or apply operation. Unlike regular resources (which are persisted to state) or data sources (which store their values in state), ephemeral resources are never written to the state file. They are ideal for short-lived credentials, temporary tokens, and other sensitive values.

## The Three Resource Types

```hcl
# 1. Regular resource: creates and manages infrastructure, stored in state

resource "aws_s3_bucket" "app" {
  bucket = "myapp-data"
  # Full resource data stored in state
}

# 2. Data source: reads existing data, stored in state
data "aws_vpc" "main" {
  id = var.vpc_id
  # Read values stored in state (security concern for sensitive data)
}

# 3. Ephemeral resource: fetches values, NEVER stored in state
ephemeral "aws_secretsmanager_secret_version" "db_pass" {
  secret_id = "myapp/db-password"
  # Values available during apply but never written to state
}
```

## When Ephemeral Resources Are Evaluated

```hcl
# Ephemeral resources follow this lifecycle:
# 1. Open: Called at the start of plan or apply to fetch the value
# 2. Use: Value is available to reference in configuration
# 3. Close: Called at the end of the operation to clean up / release

ephemeral "vault_generic_secret" "app" {
  path = "secret/myapp/config"
  # Opened before any resource that references it
  # Closed after all referencing operations complete
}

# Resources that use ephemeral values are processed
# while the ephemeral resource is still "open"
resource "aws_db_instance" "main" {
  password = jsondecode(ephemeral.vault_generic_secret.app.data_json).db_password
}
```

## Ephemeral Resource Providers

```hcl
# Providers that support ephemeral resources:
# - AWS (aws_secretsmanager_secret_version, aws_eks_cluster_auth)
# - Vault (vault_kv_secret_v2, vault_aws_access_credentials, etc.)
# - TLS (tls_private_key)
# - Kubernetes (kubernetes_token_request)

# Example: AWS Secrets Manager
ephemeral "aws_secretsmanager_secret_version" "config" {
  secret_id = "myapp/config"
}

# Example: Vault KV v2
ephemeral "vault_kv_secret_v2" "creds" {
  mount = "secret"
  name  = "myapp/credentials"
}

# Example: TLS key generation
ephemeral "tls_private_key" "server" {
  algorithm = "RSA"
  rsa_bits  = 4096
}
```

## What Can Reference Ephemeral Values

```hcl
ephemeral "aws_secretsmanager_secret_version" "secret" {
  secret_id = "myapp/secret"
}

# Resources can use ephemeral values in write-only attributes
resource "aws_db_instance" "main" {
  password = jsondecode(ephemeral.aws_secretsmanager_secret_version.secret.secret_string).password
}

# Ephemeral locals can reference ephemeral values
locals {
  ephemeral db_password = jsondecode(
    ephemeral.aws_secretsmanager_secret_version.secret.secret_string
  ).password
}

# Provider configurations can use ephemeral values
provider "github" {
  token = ephemeral.aws_secretsmanager_secret_version.secret.secret_string
}
```

## Ephemeral Values Cannot Flow to State

```hcl
ephemeral "tls_private_key" "app" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

# ERROR: Cannot store ephemeral value in regular resource
# resource "aws_ssm_parameter" "key" {
#   value = ephemeral.tls_private_key.app.private_key_pem  # Error!
# }

# Correct: Use write-only attribute if available
resource "aws_db_instance" "main" {
  password = ephemeral.some_secret.value  # OK - password is write-only
}

# Or: Store derived non-sensitive data
resource "aws_ssm_parameter" "key_public" {
  value = ephemeral.tls_private_key.app.public_key_pem  # OK - public key is not sensitive
}
```

## Difference from Sensitive Values

```hcl
# sensitive variable: stored in state but redacted from display
variable "db_password" {
  sensitive = true
  # Still in state file - just not shown in terminal output
}

# ephemeral resource: never stored in state
ephemeral "aws_secretsmanager_secret_version" "db_pass" {
  secret_id = "myapp/db-pass"
  # NEVER in state file - stronger security guarantee
}
```

## Re-evaluation on Every Apply

```hcl
# Unlike data sources (which can be cached), ephemeral resources
# are re-evaluated on every plan and apply

ephemeral "vault_aws_access_credentials" "deploy" {
  backend = "aws"
  role    = "deploy"
  # Gets fresh credentials on EVERY run
  # Works correctly with credential rotation
}
```

## Conclusion

Ephemeral resources represent the most secure way to handle sensitive values in OpenTofu. By never writing values to state, they eliminate the risk of secrets appearing in state files, remote state backends, or audit logs. They are re-fetched on every apply, making them naturally compatible with credential rotation. Use ephemeral resources for passwords, API keys, temporary tokens, private keys, and any other sensitive value that should not persist beyond a single deployment operation. As OpenTofu providers add support for ephemeral resources, they will become the standard approach for secrets management in infrastructure as code.
