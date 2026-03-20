# How to Use Ephemeral Resources Introduced in OpenTofu 1.11

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Ephemeral Resources, OpenTofu 1.11, Secrets Management, Infrastructure as Code

Description: Learn how to use ephemeral resources introduced in OpenTofu 1.11 to retrieve short-lived credentials and secrets without storing them in state.

## Introduction

OpenTofu 1.11 introduced ephemeral resources - a new resource kind that is opened and closed during a plan or apply operation but never written to state. This makes them ideal for retrieving short-lived credentials, rotating secrets, or accessing values that should never be persisted.

## Declaring an Ephemeral Resource

Ephemeral resources use the `ephemeral` block instead of `resource`.

```hcl
# Retrieve an AWS Secrets Manager secret without storing it in state

ephemeral "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/myapp/db-password"
}

# Use the secret value in a resource
resource "aws_db_instance" "main" {
  identifier = "myapp-prod"
  engine     = "postgres"
  # The password is retrieved ephemerally and never written to state
  password   = ephemeral.aws_secretsmanager_secret_version.db_password.secret_string
}
```

## Ephemeral vs Regular Data Sources

Regular `data` sources are stored in state; ephemeral resources are not.

```hcl
# Regular data source - value IS stored in state (avoid for secrets)
data "aws_secretsmanager_secret_version" "api_key" {
  secret_id = "prod/myapp/api-key"
}

# Ephemeral resource - value is NOT stored in state (preferred for secrets)
ephemeral "aws_secretsmanager_secret_version" "api_key" {
  secret_id = "prod/myapp/api-key"
}
```

## Ephemeral Credentials for Provider Configuration

Use ephemeral resources to configure providers with short-lived credentials.

```hcl
ephemeral "aws_sts_assume_role" "cross_account" {
  role_arn    = "arn:aws:iam::999888777666:role/DeployRole"
  session_name = "opentofu-deploy"
}

provider "aws" {
  alias      = "cross_account"
  region     = "us-east-1"
  access_key = ephemeral.aws_sts_assume_role.cross_account.access_key_id
  secret_key = ephemeral.aws_sts_assume_role.cross_account.secret_access_key
  token      = ephemeral.aws_sts_assume_role.cross_account.session_token
}
```

## Retrieving Vault Secrets Ephemerally

Fetch HashiCorp Vault secrets without leaving traces in state.

```hcl
ephemeral "vault_kv_secret_v2" "app_config" {
  mount = "secret"
  name  = "myapp/prod"
}

resource "kubernetes_secret" "app_config" {
  metadata {
    name      = "myapp-config"
    namespace = "production"
  }

  data = {
    database_url = ephemeral.vault_kv_secret_v2.app_config.data["database_url"]
    api_key      = ephemeral.vault_kv_secret_v2.app_config.data["api_key"]
  }
}
```

## Ephemeral Resource Lifecycle

Ephemeral resources have a distinct lifecycle compared to regular resources.

```hcl
Plan/Apply lifecycle:

1. OpenTofu opens ephemeral resource (calls provider's OpenEphemeralResource)
2. Ephemeral value is available in memory for the duration of the operation
3. OpenTofu uses the value to configure resources and providers
4. Operation completes → OpenTofu closes the ephemeral resource
5. Value is discarded - never written to state file

This means:
- No secret leakage into .tfstate files
- No risk of secrets in state backups
- Credentials can be truly short-lived (valid only during apply)
```

## Using Ephemeral Resources in Check Blocks

Verify access to ephemeral resources in health checks.

```hcl
ephemeral "aws_secretsmanager_secret_version" "health_check" {
  secret_id = "prod/myapp/health-check-token"
}

check "secret_accessible" {
  assert {
    condition     = ephemeral.aws_secretsmanager_secret_version.health_check.secret_string != ""
    error_message = "Health check secret is not accessible"
  }
}
```

## Summary

Ephemeral resources in OpenTofu 1.11 solve a long-standing problem with secrets management in IaC - sensitive values leaking into state files. By using the `ephemeral` block for secrets, credentials, and short-lived tokens, you keep your state clean while still using dynamic values during plan and apply. This is the preferred pattern for any value that should never be persisted.
