# How to Pass Backend Credentials via Environment Variables in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Security, State Management

Description: Learn how to provide OpenTofu backend credentials securely through environment variables, avoiding hardcoded secrets in configuration files.

## Introduction

Every OpenTofu backend supports passing credentials via environment variables. This is the recommended approach for CI/CD pipelines and production environments - it keeps secrets out of your `.tf` files and version control, and integrates naturally with secrets managers and CI/CD secret injection.

## S3 Backend Environment Variables

```bash
# Standard AWS credential variables (used by S3 backend)

export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_SESSION_TOKEN="FwoGZXIvYXdzEJr..."  # For temporary credentials
export AWS_REGION="us-east-1"
export AWS_DEFAULT_REGION="us-east-1"

# Profile-based authentication
export AWS_PROFILE="terraform-prod"

# S3-specific backend variables
export AWS_S3_BUCKET="my-terraform-state"  # Alternative to inline config
```

```hcl
# backend.tf - no credentials needed with env vars
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
    # Credentials come from environment
  }
}
```

## Azure Backend Environment Variables

```bash
export ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
export ARM_CLIENT_SECRET="your-client-secret"
export ARM_TENANT_ID="00000000-0000-0000-0000-000000000000"
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"

# For Managed Identity
export ARM_USE_MSI=true

# For OIDC (GitHub Actions, etc.)
export ARM_USE_OIDC=true
export ARM_OIDC_TOKEN=$(cat /path/to/oidc-token)
```

## GCS Backend Environment Variables

```bash
# Application Default Credentials
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"

# Or inline JSON
export GOOGLE_CREDENTIALS=$(cat service-account.json)

# Project and region
export GOOGLE_PROJECT="my-project"
export GOOGLE_REGION="us-central1"
```

## Consul Backend Environment Variables

```bash
export CONSUL_HTTP_ADDR="https://consul.example.com:8500"
export CONSUL_HTTP_TOKEN="your-acl-token"
export CONSUL_HTTP_SSL_VERIFY="true"
export CONSUL_CACERT="/etc/consul/certs/ca.pem"
export CONSUL_CLIENT_CERT="/etc/consul/certs/client.pem"
export CONSUL_CLIENT_KEY="/etc/consul/certs/client-key.pem"
```

## PostgreSQL Backend Environment Variables

```bash
export PG_CONN_STR="postgresql://user:password@host:5432/dbname?sslmode=require"
```

## HTTP Backend Environment Variables

```bash
export TF_HTTP_USERNAME="terraform-user"
export TF_HTTP_PASSWORD="your-api-token"
export TF_HTTP_ADDRESS="https://state-server.example.com/state/prod"
```

## Kubernetes Backend Environment Variables

```bash
export KUBE_CONFIG_PATH="~/.kube/config"
export KUBE_CONTEXT="prod-cluster"
export KUBE_TOKEN="eyJhbGciOiJSUzI1NiI..."
```

## Loading Credentials from Secrets Managers

### AWS Secrets Manager

```bash
#!/bin/bash
# Load credentials from Secrets Manager

CREDS=$(aws secretsmanager get-secret-value \
  --secret-id "terraform/backend-credentials" \
  --query 'SecretString' \
  --output text)

export TF_VAR_backend_access_key=$(echo $CREDS | jq -r '.access_key')
export TF_VAR_backend_secret_key=$(echo $CREDS | jq -r '.secret_key')
```

### HashiCorp Vault

```bash
#!/bin/bash
# Load credentials from Vault

export VAULT_ADDR="https://vault.example.com"
export VAULT_TOKEN=$(vault login -method=aws -token-only)

# Get backend credentials
CREDS=$(vault kv get -format=json secret/terraform/backend-creds)
export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.data.data.access_key_id')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.data.data.secret_access_key')
```

### GCP Secret Manager

```bash
#!/bin/bash
# Load GCS credentials from Secret Manager

SA_KEY=$(gcloud secrets versions access latest \
  --secret="opentofu-sa-key" \
  --project="my-project")

export GOOGLE_CREDENTIALS="$SA_KEY"
```

## CI/CD Integration Pattern

```yaml
# GitHub Actions
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions
          aws-region: us-east-1
        # This sets AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN

      - name: OpenTofu Init
        run: tofu init  # Uses the AWS env vars from above
```

## Conclusion

Environment variables are the standard, secure mechanism for passing backend credentials to OpenTofu. Every major backend supports them, they integrate cleanly with CI/CD secret management, and they prevent credentials from being committed to version control. Combine environment variables with secrets managers (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault) for a complete, auditable credential management solution.
