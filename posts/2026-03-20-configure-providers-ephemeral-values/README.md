# How to Configure Providers with Ephemeral Values in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Ephemeral Resources, Provider Configuration, Secrets, HCL, Infrastructure as Code

Description: Learn how to use ephemeral values to configure providers in OpenTofu so that credentials and tokens are never stored in state files.

---

Provider configurations often need credentials like API tokens, passwords, or access keys. Using ephemeral resources to supply these values ensures that sensitive credentials are used at runtime but never written to the state file.

---

## The Problem: Credentials in Provider Configs

Provider credentials specified directly or through variables end up in state:

```hcl
# BAD: API token could end up in state or plan output
provider "github" {
  token = var.github_token   # stored in state as part of provider config hash
}
```

With ephemeral values, the credential is fetched at runtime and discarded.

---

## GitHub Provider with Ephemeral Token

```hcl
# Fetch the GitHub token from AWS Parameter Store ephemerally
ephemeral "aws_ssm_parameter" "github_token" {
  name            = "/production/github/token"
  with_decryption = true
}

# Configure the provider with the ephemeral value
provider "github" {
  token = ephemeral.aws_ssm_parameter.github_token.value
  # The token is used to configure the provider but never stored in state
}
```

---

## Kubernetes Provider with Ephemeral Credentials

```hcl
# Fetch kubeconfig credentials from Secrets Manager
ephemeral "aws_secretsmanager_secret_version" "kubeconfig" {
  secret_id = "production/kubernetes/admin-credentials"
}

locals {
  k8s_creds = jsondecode(ephemeral.aws_secretsmanager_secret_version.kubeconfig.secret_string)
}

provider "kubernetes" {
  host                   = local.k8s_creds.host
  client_certificate     = base64decode(local.k8s_creds.client_certificate)
  client_key             = base64decode(local.k8s_creds.client_key)
  cluster_ca_certificate = base64decode(local.k8s_creds.cluster_ca)
}
```

---

## Database Provider with Vault Dynamic Credentials

```hcl
# Get short-lived database credentials from Vault
ephemeral "vault_database_secret" "pg_creds" {
  mount = "database"
  name  = "postgresql-production"
}

provider "postgresql" {
  host     = aws_db_instance.main.address
  port     = aws_db_instance.main.port
  database = "app"
  username = ephemeral.vault_database_secret.pg_creds.username
  password = ephemeral.vault_database_secret.pg_creds.password
  # Credentials are ephemeral — not stored in state
}
```

---

## Cross-Account AWS Provider

```hcl
# Assume a role and get temporary credentials
ephemeral "aws_iam_role" "production_access" {
  role_arn     = "arn:aws:iam::123456789012:role/OpenTofuDeployRole"
  session_name = "opentofu-${var.run_id}"
  duration     = "30m"
}

# Configure an aliased provider for the production account
provider "aws" {
  alias  = "production"
  region = "us-east-1"

  access_key = ephemeral.aws_iam_role.production_access.access_key_id
  secret_key = ephemeral.aws_iam_role.production_access.secret_access_key
  token      = ephemeral.aws_iam_role.production_access.session_token
}

# Deploy into production using the temporary credentials
resource "aws_s3_bucket" "deploy" {
  provider = aws.production
  bucket   = "production-deployments"
}
```

---

## DataDog Provider with Ephemeral API Key

```hcl
ephemeral "aws_secretsmanager_secret_version" "datadog" {
  secret_id = "monitoring/datadog/api-keys"
}

locals {
  dd_secrets = jsondecode(ephemeral.aws_secretsmanager_secret_version.datadog.secret_string)
}

provider "datadog" {
  api_key = local.dd_secrets.api_key
  app_key = local.dd_secrets.app_key
}
```

---

## Ephemeral Values in provider_meta

Some providers accept configuration in a `provider_meta` block — these can also use ephemeral values:

```hcl
ephemeral "aws_ssm_parameter" "license_key" {
  name = "/enterprise/license-key"
  with_decryption = true
}

terraform {
  required_providers {
    someenterprise = {
      source  = "enterprise/someenterprise"
      version = "~> 2.0"
    }
  }
}

provider "someenterprise" {
  license_key = ephemeral.aws_ssm_parameter.license_key.value
}
```

---

## Why This Matters

State files are often stored in S3 or remote backends with broad access. Any value stored in state is accessible to anyone who can read the state. By using ephemeral values in provider configurations, you ensure:

1. Provider credentials are never written to state files
2. Temporary credentials expire naturally
3. Audit trails show credential fetches from the secrets store
4. Rotating secrets doesn't require state updates

---

## Summary

Use ephemeral resources to supply provider credentials dynamically — API tokens, database passwords, temporary AWS credentials, and TLS certificates. The provider uses the value during the operation, but it is never written to the state file. This is the recommended approach for any sensitive provider configuration value in OpenTofu 1.10+.
