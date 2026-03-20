# How to Read Vault Secrets in OpenTofu Configurations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Vault, Secrets, Security, Secret Management

Description: Learn how to read HashiCorp Vault secrets in OpenTofu configurations using the Vault provider data sources to inject sensitive values without storing them in state or version control.

## Introduction

OpenTofu's Vault provider allows reading secrets directly from HashiCorp Vault during plan and apply. This keeps sensitive values out of `.tfvars` files and version control, while still making them available as inputs to resources.

## Provider Configuration

```hcl
# versions.tf

terraform {
  required_providers {
    vault = {
      source  = "hashicorp/vault"
      version = "~> 4.0"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# provider.tf
provider "vault" {
  address = "https://vault.example.com:8200"
  # Token from VAULT_TOKEN environment variable
}
```

## Reading KV v2 Secrets

```hcl
# Read a secret from the KV v2 secrets engine
data "vault_kv_secret_v2" "database" {
  mount = "secret"
  name  = "prod/database"
}

resource "aws_db_instance" "main" {
  identifier     = "prod-db"
  engine         = "postgres"
  instance_class = "db.r6g.large"

  username = data.vault_kv_secret_v2.database.data["username"]
  password = data.vault_kv_secret_v2.database.data["password"]
  db_name  = data.vault_kv_secret_v2.database.data["database_name"]

  # ... other configuration
}
```

## Reading KV v1 Secrets

```hcl
data "vault_kv_secret" "app_config" {
  path = "kv/prod/application"
}

locals {
  api_key      = data.vault_kv_secret.app_config.data["api_key"]
  webhook_url  = data.vault_kv_secret.app_config.data["webhook_url"]
}
```

## Reading Generic Secrets

```hcl
# For custom secrets engines
data "vault_generic_secret" "tls_cert" {
  path = "pki/issue/web-server"
}

resource "aws_acm_certificate_import" "web" {
  certificate_body  = data.vault_generic_secret.tls_cert.data["certificate"]
  private_key       = data.vault_generic_secret.tls_cert.data["private_key"]
  certificate_chain = data.vault_generic_secret.tls_cert.data["ca_chain"]
}
```

## Injecting Secrets into ECS Task Definitions

```hcl
data "vault_kv_secret_v2" "app_secrets" {
  mount = "secret"
  name  = "prod/app"
}

resource "aws_ecs_task_definition" "app" {
  family = "app"

  container_definitions = jsonencode([
    {
      name  = "app"
      image = "123456789012.dkr.ecr.us-east-1.amazonaws.com/app:latest"

      environment = [
        {
          name  = "DATABASE_URL"
          value = "postgres://${data.vault_kv_secret_v2.app_secrets.data["db_user"]}:${data.vault_kv_secret_v2.app_secrets.data["db_pass"]}@${aws_db_instance.main.endpoint}/appdb"
        },
        {
          name  = "REDIS_URL"
          value = data.vault_kv_secret_v2.app_secrets.data["redis_url"]
        }
      ]
    }
  ])
}
```

## Preventing Secrets from Appearing in Plan Output

```hcl
# Mark outputs as sensitive to prevent them from appearing in logs
output "db_password" {
  value     = data.vault_kv_secret_v2.database.data["password"]
  sensitive = true
}

# Use the nonsensitive() function carefully - only when you need to pass
# a sensitive value to a resource that doesn't accept sensitive types
resource "some_resource" "example" {
  value = nonsensitive(data.vault_kv_secret_v2.config.data["some_value"])
}
```

## Caching Vault Reads

```hcl
# Vault data sources are read on every plan/apply
# Use locals to read once and reference multiple times
locals {
  db_creds = data.vault_kv_secret_v2.database.data
}

resource "aws_db_instance" "primary" {
  username = local.db_creds["username"]
  password = local.db_creds["password"]
}

resource "aws_db_instance" "replica" {
  username = local.db_creds["username"]
  password = local.db_creds["password"]
}
```

## Conclusion

Reading Vault secrets in OpenTofu configurations keeps sensitive values out of state files and version control when combined with proper state encryption. The `vault_kv_secret_v2` data source is the most common approach, but the `vault_generic_secret` data source works with any Vault secrets engine. Always mark secret outputs as `sensitive = true` to prevent accidental exposure in logs.
