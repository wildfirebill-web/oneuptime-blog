# How to Configure the Cloud Backend in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Cloud Backend, Terraform Cloud, Remote State, Configuration

Description: Learn how to configure the cloud backend in OpenTofu to use Terraform Cloud or HCP Terraform for remote state storage, plan execution, and workspace management.

## Introduction

The cloud backend in OpenTofu connects your configurations to Terraform Cloud (or HCP Terraform), providing remote state storage, plan/apply execution, a collaborative UI, and policy enforcement. It replaces the older `remote` backend with a more feature-rich integration. OpenTofu maintains compatibility with the Terraform Cloud API.

## Basic Cloud Backend Configuration

```hcl
# main.tf

terraform {
  cloud {
    organization = "my-company"

    workspaces {
      name = "production-infrastructure"
    }
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

## Authentication

```bash
# Option 1: Interactive login

tofu login

# This opens a browser, prompts for a Terraform Cloud token,
# and saves it to ~/.terraform.d/credentials.tfrc.json

# Option 2: Environment variable
export TF_TOKEN_app_terraform_io="your-api-token"

# Option 3: Direct CLI config
cat > ~/.terraform.d/credentials.tfrc.json << 'EOF'
{
  "credentials": {
    "app.terraform.io": {
      "token": "your-api-token-here"
    }
  }
}
EOF
```

## Workspace Tag-Based Selection

```hcl
# Instead of a single workspace, select by tags
terraform {
  cloud {
    organization = "my-company"

    workspaces {
      tags = ["aws", "production"]
    }
  }
}
```

```bash
# When using tags, tofu selects the workspace interactively on init
tofu init

# Or specify via TF_WORKSPACE environment variable
export TF_WORKSPACE="production-us-east-1"
tofu init
```

## Hostname for HCP Terraform or TFE

```hcl
# For Terraform Enterprise (self-hosted)
terraform {
  cloud {
    hostname     = "tfe.internal.company.com"
    organization = "my-company"

    workspaces {
      name = "production"
    }
  }
}

# credentials block for custom hostname
# ~/.terraform.d/credentials.tfrc.json
# {
#   "credentials": {
#     "tfe.internal.company.com": {
#       "token": "your-tfe-token"
#     }
#   }
# }
```

## Variable Configuration

```hcl
# Variables can be set in the workspace UI or via API
# Environment variables and Terraform variables

variable "environment" {
  type    = string
  default = "production"
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}
```

```bash
# Set workspace variables via Terraform Cloud API
curl -X POST \
  -H "Authorization: Bearer $TF_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/workspaces/$WORKSPACE_ID/vars" \
  -d '{
    "data": {
      "type": "vars",
      "attributes": {
        "key": "AWS_ACCESS_KEY_ID",
        "value": "'$AWS_ACCESS_KEY_ID'",
        "category": "env",
        "sensitive": true
      }
    }
  }'
```

## Initializing with Cloud Backend

```bash
# Initialize (prompts for workspace selection if using tags)
tofu init

# Expected output:
# Initializing Terraform Cloud...
# Terraform Cloud has been successfully initialized!
#
# You may now begin working with Terraform Cloud.
# The current workspace is "production-infrastructure".

# Show current workspace
tofu workspace show
```

## Workspace Creation

```bash
# Create a new workspace via Terraform Cloud API
curl -X POST \
  -H "Authorization: Bearer $TF_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/organizations/my-company/workspaces" \
  -d '{
    "data": {
      "type": "workspaces",
      "attributes": {
        "name": "production-infrastructure",
        "terraform-version": "1.7.0",
        "auto-apply": false,
        "working-directory": ""
      }
    }
  }'
```

## .terraformignore for Cloud Backend

```text
# .terraformignore - exclude files from cloud backend uploads
.git/
.terraform/
*.tfstate
*.tfstate.backup
.DS_Store
*.log
tests/
.terragrunt-cache/
```

## Migrating from Local to Cloud Backend

```bash
# Step 1: Add cloud backend configuration to main.tf
# (as shown above)

# Step 2: Run init with migration
tofu init

# OpenTofu detects existing local state and prompts:
# "Would you like to copy existing state to the new backend? (yes/no)"
# Answer: yes

# Step 3: Verify state was migrated
tofu state list  # Should show same resources
```

## Conclusion

The cloud backend configuration requires three elements: the `organization` name, `workspaces` configuration (either a single name or a tag set), and authentication via token. The `tofu login` command handles interactive authentication and token storage. For CI/CD, set the `TF_TOKEN_app_terraform_io` environment variable. After initialization, all `tofu plan` and `tofu apply` commands by default execute remotely in Terraform Cloud, with logs streamed to the local terminal.
