# How to Use Ephemeral Resources for Temporary Credentials in OpenTofu - Opentofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Ephemeral Resources, Temporary Credentials, Secrets, Infrastructure as Code, DevOps

Description: A guide to using ephemeral resources in OpenTofu to obtain and use temporary credentials without persisting sensitive values in state.

## Introduction

Ephemeral resources in OpenTofu are a special type of resource that exist only during the plan/apply lifecycle and are never written to state. They are ideal for obtaining temporary credentials, short-lived tokens, and other sensitive values that should not persist beyond a single operation.

## What Makes Ephemeral Resources Different

```hcl
# Regular data source: values stored in state

data "aws_secretsmanager_secret_version" "db_pass" {
  secret_id = "myapp/db-password"
  # Value is stored in terraform.tfstate - SECURITY RISK
}

# Ephemeral resource: values NOT stored in state
ephemeral "aws_secretsmanager_secret_version" "db_pass" {
  secret_id = "myapp/db-password"
  # Value is used during apply but never written to state
}
```

## Vault Temporary Credentials

```hcl
# Generate temporary AWS credentials from Vault
ephemeral "vault_aws_access_credentials" "deploy" {
  backend = "aws"
  role    = "deploy-role"
  ttl     = "1h"
}

# Use temporary credentials to configure AWS provider
provider "aws" {
  alias      = "temporary"
  access_key = ephemeral.vault_aws_access_credentials.deploy.access_key
  secret_key = ephemeral.vault_aws_access_credentials.deploy.secret_key
  token      = ephemeral.vault_aws_access_credentials.deploy.security_token
  region     = var.region
}

resource "aws_s3_bucket" "deploy" {
  provider = aws.temporary
  bucket   = "deploy-artifacts-${var.environment}"
}
```

## Database Password from Secrets Manager

```hcl
# Fetch DB password ephemerally - not stored in state
ephemeral "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "myapp/${var.environment}/db-password"
}

resource "aws_db_instance" "main" {
  identifier = "myapp-${var.environment}"
  engine     = "postgres"

  # Password used during creation but not stored in state
  password = jsondecode(
    ephemeral.aws_secretsmanager_secret_version.db_password.secret_string
  ).password

  lifecycle {
    ignore_changes = [password]  # Don't track password changes in state
  }
}
```

## Kubernetes Service Account Token

```hcl
# Ephemeral Kubernetes token for authentication
ephemeral "kubernetes_token_request" "deploy" {
  metadata {
    name      = "deploy-token"
    namespace = "default"
  }

  spec {
    audiences          = ["https://kubernetes.default.svc"]
    expiration_seconds = 3600
  }
}

# Use token to authenticate external service registration
resource "consul_service" "k8s_app" {
  name    = "my-app"
  address = var.cluster_endpoint
  port    = 443

  meta = {
    # Token available during apply but not in state
    auth_token = ephemeral.kubernetes_token_request.deploy.token
  }
}
```

## SSH Keys for Provisioning

```hcl
# Generate ephemeral SSH key for one-time provisioning
ephemeral "tls_private_key" "provisioner" {
  algorithm = "ED25519"
}

# Add public key to instance for provisioning
resource "aws_key_pair" "provisioner" {
  key_name   = "provisioner-${var.deployment_id}"
  public_key = ephemeral.tls_private_key.provisioner.public_key_openssh
}

resource "aws_instance" "web" {
  ami      = var.ami_id
  key_name = aws_key_pair.provisioner.key_name

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = ephemeral.tls_private_key.provisioner.private_key_openssh
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
    ]
  }
}
```

## OIDC Tokens for CI/CD

```hcl
# Get ephemeral OIDC token for authentication
ephemeral "github_actions_oidc_token" "aws" {
  audience = "sts.amazonaws.com"
}

# Configure AWS provider with OIDC (no long-lived credentials)
provider "aws" {
  region = var.region

  assume_role_with_web_identity {
    role_arn                = "arn:aws:iam::${var.account_id}:role/GitHubActionsRole"
    web_identity_token      = ephemeral.github_actions_oidc_token.aws.token
    session_name            = "github-actions-deploy"
  }
}
```

## Checking State for Sensitive Data

```bash
# With regular data sources, sensitive values appear in state
cat terraform.tfstate | grep -i password  # DANGEROUS

# With ephemeral resources, no sensitive data in state
# Safe to inspect state file
tofu state show aws_db_instance.main
# password field will show null or be absent
```

## Ephemeral Resource Lifecycle

```hcl
# Ephemeral resources:
# 1. Are fetched/created at the start of plan/apply
# 2. Their values are available during the operation
# 3. Are explicitly closed/released at the end of the operation
# 4. Are NEVER written to the state file
# 5. Are re-fetched on every plan/apply (not cached)

ephemeral "vault_kv_secret_v2" "app_secrets" {
  mount = "secret"
  name  = "myapp/${var.environment}"
  # Values available during this apply
  # Will be re-fetched on next apply
}
```

## Conclusion

Ephemeral resources solve the fundamental security problem with traditional data sources: sensitive values being persisted to state files. By using ephemeral resources for temporary credentials, API tokens, and passwords, you ensure that these values are only available during the apply operation and are never written to disk. This makes your infrastructure deployments more secure, especially in environments where state files are stored remotely and accessible to multiple team members. Use ephemeral resources whenever you need to use sensitive credentials to configure resources but don't want those credentials to persist in state.
