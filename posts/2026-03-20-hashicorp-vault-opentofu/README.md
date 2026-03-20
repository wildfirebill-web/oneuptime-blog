# How to Integrate HashiCorp Vault with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, HashiCorp Vault, Secrets Management, Security, Infrastructure as Code

Description: Learn how to use the Vault provider in OpenTofu to dynamically generate cloud credentials, read static secrets, and avoid hardcoding any sensitive values in your configurations.

## Introduction

HashiCorp Vault is the industry-standard secrets management platform. The OpenTofu Vault provider lets you read secrets and generate dynamic cloud credentials directly within your configurations, ensuring that real credentials never touch configuration files or CI/CD environment variables.

## Configuring the Vault Provider

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

provider "vault" {
  address = "https://vault.example.com:8200"
  # Vault token can be set via VAULT_TOKEN environment variable
  # For CI/CD, use AppRole or AWS auth method instead
}
```

## Reading KV Secrets

```hcl
# Read a static secret from Vault KV v2
data "vault_kv_secret_v2" "db_creds" {
  mount = "secret"
  name  = "prod/database"
}

# Use the secret in a resource - values are marked sensitive automatically
resource "aws_db_instance" "main" {
  identifier = "prod-postgres"
  engine     = "postgres"
  username   = data.vault_kv_secret_v2.db_creds.data["username"]
  password   = data.vault_kv_secret_v2.db_creds.data["password"]
  # ...
}
```

## Dynamic AWS Credentials via Vault AWS Secrets Engine

This is the most powerful pattern - Vault generates short-lived AWS credentials on demand:

```hcl
# Generate temporary AWS credentials for the apply session
data "vault_aws_access_credentials" "aws_creds" {
  backend = "aws"
  role    = "opentofu-deploy-role"
  type    = "iam_user"
}

provider "aws" {
  access_key = data.vault_aws_access_credentials.aws_creds.access_key
  secret_key = data.vault_aws_access_credentials.aws_creds.secret_key
  region     = "us-east-1"
}
```

## AppRole Authentication for CI/CD

For pipelines, use the AppRole auth method instead of a long-lived token:

```hcl
provider "vault" {
  address   = "https://vault.example.com:8200"
  auth_login {
    path = "auth/approle/login"
    parameters = {
      role_id   = var.vault_role_id
      secret_id = var.vault_secret_id
    }
  }
}
```

```bash
# In your CI pipeline, inject role_id and secret_id as environment variables
export TF_VAR_vault_role_id="${VAULT_ROLE_ID}"
export TF_VAR_vault_secret_id="${VAULT_SECRET_ID}"
tofu apply
```

## Writing Secrets to Vault

OpenTofu can also write generated secrets back to Vault for consumption by applications:

```hcl
# Generate a random password and store it in Vault
resource "random_password" "db" {
  length  = 32
  special = true
}

resource "vault_kv_secret_v2" "db_password" {
  mount = "secret"
  name  = "prod/generated/db-password"

  data_json = jsonencode({
    password = random_password.db.result
  })
}

# Use the generated password in the database resource
resource "aws_db_instance" "main" {
  password = random_password.db.result
}
```

## Vault Policies for OpenTofu

Create a least-privilege Vault policy for the OpenTofu AppRole:

```hcl
# In Vault (using vault_policy resource)
resource "vault_policy" "opentofu_deploy" {
  name = "opentofu-deploy"
  policy = <<EOF
# Allow reading database credentials
path "secret/data/prod/database" {
  capabilities = ["read"]
}

# Allow generating AWS credentials
path "aws/creds/opentofu-deploy-role" {
  capabilities = ["read"]
}

# Allow writing generated secrets
path "secret/data/prod/generated/*" {
  capabilities = ["create", "update"]
}
EOF
}
```

## Conclusion

Integrating Vault with OpenTofu eliminates static credentials entirely. By generating dynamic, short-lived cloud credentials via the AWS secrets engine and reading application secrets via KV data sources, your OpenTofu configurations contain no sensitive values - and every credential issued has an audit trail in Vault.
