# How to Configure Providers with Ephemeral Values in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Providers, Ephemeral Values, Secrets, Infrastructure as Code, DevOps

Description: A guide to configuring OpenTofu providers using ephemeral values to avoid storing provider credentials in state.

## Introduction

OpenTofu providers often need sensitive credentials like API keys, tokens, and passwords. Configuring providers with ephemeral values ensures these credentials are used during plan/apply but never written to the state file. This improves security when state is stored remotely or shared across teams.

## Provider Configuration with Ephemeral Secrets

```hcl
# Fetch API key ephemerally
ephemeral "aws_secretsmanager_secret_version" "github_token" {
  secret_id = "platform/github-token"
}

# Configure provider without persisting the token
provider "github" {
  token = ephemeral.aws_secretsmanager_secret_version.github_token.secret_string
  # Token used during apply but never written to state
}
```

## Multi-Provider Setup with Vault

```hcl
# Fetch credentials for multiple providers from Vault
ephemeral "vault_kv_secret_v2" "credentials" {
  mount = "secret"
  name  = "platform/provider-credentials"
}

locals {
  creds = jsondecode(ephemeral.vault_kv_secret_v2.credentials.data_json)
}

provider "datadog" {
  api_key = local.creds.datadog_api_key
  app_key = local.creds.datadog_app_key
}

provider "pagerduty" {
  token = local.creds.pagerduty_token
}

provider "slack" {
  token = local.creds.slack_bot_token
}
```

## AWS Provider with Assumed Role

```hcl
# Get Vault-issued temporary AWS credentials
ephemeral "vault_aws_access_credentials" "infra" {
  backend = "aws"
  role    = "infrastructure-deployer"
}

provider "aws" {
  region     = var.region
  access_key = ephemeral.vault_aws_access_credentials.infra.access_key
  secret_key = ephemeral.vault_aws_access_credentials.infra.secret_key
  token      = ephemeral.vault_aws_access_credentials.infra.security_token
}

# All AWS resources created with these ephemeral credentials
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

## Kubernetes Provider with Ephemeral Token

```hcl
# Get EKS cluster auth ephemerally
ephemeral "aws_eks_cluster_auth" "main" {
  name = aws_eks_cluster.main.name
}

provider "kubernetes" {
  host                   = aws_eks_cluster.main.endpoint
  cluster_ca_certificate = base64decode(
    aws_eks_cluster.main.certificate_authority[0].data
  )
  # Token not stored in state
  token = ephemeral.aws_eks_cluster_auth.main.token
}

provider "helm" {
  kubernetes {
    host                   = aws_eks_cluster.main.endpoint
    cluster_ca_certificate = base64decode(
      aws_eks_cluster.main.certificate_authority[0].data
    )
    token = ephemeral.aws_eks_cluster_auth.main.token
  }
}
```

## Database Provider with Ephemeral Password

```hcl
# Get database password from Secrets Manager
ephemeral "aws_secretsmanager_secret_version" "db_admin" {
  secret_id = "myapp/db-admin-credentials"
}

locals {
  db_creds = jsondecode(
    ephemeral.aws_secretsmanager_secret_version.db_admin.secret_string
  )
}

provider "postgresql" {
  host     = aws_db_instance.main.address
  port     = aws_db_instance.main.port
  database = "postgres"
  username = local.db_creds.username
  password = local.db_creds.password  # Not stored in state
  sslmode  = "require"
}

resource "postgresql_database" "app" {
  name = "myapp_${var.environment}"
}
```

## Terraform Cloud Provider

```hcl
# Fetch Terraform Cloud token from Vault
ephemeral "vault_kv_secret_v2" "tfc" {
  mount = "secret"
  name  = "platform/terraform-cloud"
}

provider "tfe" {
  token = ephemeral.vault_kv_secret_v2.tfc.data["token"]
  # Token used but not stored in state
}

resource "tfe_workspace" "app" {
  name         = "myapp-${var.environment}"
  organization = var.tfc_organization
}
```

## Environment Variables vs Ephemeral Values

```hcl
# Option 1: Environment variables (traditional approach)
# Set TF_VAR_github_token before running tofu
# provider "github" {
#   token = var.github_token  # Token in state if not marked sensitive
# }

# Option 2: Ephemeral resource (more secure)
ephemeral "aws_secretsmanager_secret_version" "github" {
  secret_id = "platform/github-token"
}

provider "github" {
  token = ephemeral.aws_secretsmanager_secret_version.github.secret_string
  # Never in state, fetched fresh each apply
}
```

## Rotating Credentials Automatically

```hcl
# Secrets Manager with automatic rotation
data "aws_secretsmanager_secret" "api_key" {
  name = "myapp/api-key"
}

ephemeral "aws_secretsmanager_secret_version" "api_key" {
  secret_id = data.aws_secretsmanager_secret.api_key.id
  # Always fetches the latest version
  # If Secrets Manager rotates the key, next apply uses new key
}

provider "external_service" {
  api_key = ephemeral.aws_secretsmanager_secret_version.api_key.secret_string
}
```

## Conclusion

Configuring providers with ephemeral values is a significant security improvement over passing credentials through variables (which may appear in state) or environment variables (which may be logged). By fetching credentials from secrets management systems like Vault or Secrets Manager at the start of each apply operation, you ensure credentials are short-lived, audited, and never written to disk. This approach works especially well with credential rotation systems, as each apply automatically picks up the latest credentials without any configuration changes.
