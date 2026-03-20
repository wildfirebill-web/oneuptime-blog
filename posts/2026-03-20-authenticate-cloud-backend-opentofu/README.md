# How to Authenticate with Cloud Backend in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Cloud Backend, Authentication, Terraform Cloud, API Tokens

Description: Learn how to authenticate OpenTofu with the Terraform Cloud backend using API tokens, environment variables, and CI/CD-specific authentication methods.

## Introduction

Authentication with the Terraform Cloud backend requires an API token that grants access to your organization's workspaces and state. OpenTofu supports multiple authentication methods: interactive login, environment variables, credentials files, and OIDC-based dynamic tokens for CI/CD. Choosing the right method depends on where OpenTofu runs.

## Authentication Methods Overview

```hcl
Method                    | Use Case
--------------------------|----------------------------------------
tofu login                | Interactive: developer machines
TF_TOKEN_* env var        | CI/CD: GitHub Actions, GitLab CI
credentials.tfrc.json     | Persistent: shared CI/CD agents
OIDC / Dynamic tokens     | GitHub Actions, HashiCorp Vault
```

## Method 1: Interactive Login

```bash
# Opens browser to generate API token

tofu login

# For Terraform Enterprise (custom hostname)
tofu login tfe.internal.company.com

# What it does:
# 1. Opens app.terraform.io in browser
# 2. Prompts you to create or use existing API token
# 3. Saves token to ~/.terraform.d/credentials.tfrc.json

# Logout
tofu logout
```

## Method 2: Environment Variables

```bash
# Standard Terraform Cloud
export TF_TOKEN_app_terraform_io="your-api-token"

# For Terraform Enterprise
export TF_TOKEN_tfe_internal_company_com="your-tfe-token"
# Note: periods in hostname are replaced with underscores

# Verify authentication
tofu init  # Should succeed without prompting for credentials
```

## Method 3: Credentials File

```json
// ~/.terraform.d/credentials.tfrc.json
{
  "credentials": {
    "app.terraform.io": {
      "token": "your-terraform-cloud-api-token"
    },
    "tfe.internal.company.com": {
      "token": "your-tfe-api-token"
    }
  }
}
```

```bash
# Also settable via CLI config file
# ~/.terraform.rc
# credentials "app.terraform.io" {
#   token = "your-token"
# }
```

## Method 4: GitHub Actions with OIDC

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # Required for OIDC
      contents: read

    steps:
      - uses: actions/checkout@v4

      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: '1.7.0'
          # Provide token directly - most common approach
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}

      - name: OpenTofu Init
        run: tofu init
```

## Method 5: Team API Tokens

```bash
# Team tokens provide access to all workspaces the team can access
# Generate via: Organization → Teams → [Team Name] → Token

# Generate programmatically via API
curl -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/teams/$TEAM_ID/authentication-token" \
  -d '{
    "data": {
      "type": "authentication-tokens"
    }
  }'

# Use the team token in CI/CD
export TF_TOKEN_app_terraform_io="team-token-value"
```

## Method 6: Organization Tokens

```bash
# Organization tokens provide full access to all workspaces
# Use for administrative scripts, not regular deployments

curl -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/organizations/my-company/authentication-token" \
  -d '{
    "data": {
      "type": "authentication-tokens"
    }
  }'
```

## Token Types and Permissions

```text
Token Type         | Scope                     | Recommended For
-------------------|---------------------------|------------------
User Token         | User's access + teams     | Developer machines
Team Token         | Team's workspaces         | CI/CD pipelines
Organization Token | All workspaces (admin)    | Admin scripts
Workspace Token    | Single workspace          | Per-workspace CI/CD
```

## Workspace-Specific Tokens

```bash
# Create a token scoped to a single workspace
curl -X POST \
  -H "Authorization: Bearer $TF_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/workspaces/$WORKSPACE_ID/authentication-token" \
  -d '{
    "data": {
      "type": "authentication-tokens"
    }
  }'

# Use workspace-scoped token (least privilege for per-workspace CI/CD)
export TF_TOKEN_app_terraform_io="workspace-scoped-token"
tofu init  # Works only for the specified workspace
```

## CI/CD Secret Management

```yaml
# GitHub Actions: store token as secret
# Settings → Secrets → Actions → New repository secret: TF_API_TOKEN

- uses: opentofu/setup-opentofu@v1
  with:
    cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}

# GitLab CI: use CI/CD variables
# Settings → CI/CD → Variables: TF_TOKEN_app_terraform_io

variables:
  TF_TOKEN_app_terraform_io: $TF_API_TOKEN

# Jenkins: use credentials binding
withCredentials([string(credentialsId: 'tf-cloud-token', variable: 'TF_TOKEN')]) {
  sh 'export TF_TOKEN_app_terraform_io=$TF_TOKEN && tofu apply'
}
```

## Verifying Authentication

```bash
# Test authentication works
tofu login  # Check if already logged in
# Expected: "The new credentials will be used for all future requests"

# Test API access directly
curl -H "Authorization: Bearer $TF_TOKEN" \
  "https://app.terraform.io/api/v2/account/details" | \
  jq '.data.attributes.username'

# Test workspace access
tofu init
tofu workspace show  # Should print workspace name without error
```

## Token Rotation

```bash
# Rotate tokens periodically for security
# 1. Generate new token in Terraform Cloud UI or API
# 2. Update secret in CI/CD system
# 3. Revoke old token

# Revoke a specific token
curl -X DELETE \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://app.terraform.io/api/v2/authentication-tokens/$TOKEN_ID"
```

## Conclusion

For developer machines, use `tofu login` to store credentials interactively. For CI/CD pipelines, use the `TF_TOKEN_app_terraform_io` environment variable set from a secrets manager - team tokens for shared pipelines, workspace-specific tokens for maximum least-privilege isolation. The `opentofu/setup-opentofu` GitHub Actions action accepts a `cli_config_credentials_token` parameter that handles credential file creation automatically, making it the simplest authentication method for GitHub Actions workflows.
