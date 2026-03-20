# How to Configure Cloud Backend Workspaces in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Cloud Backend, Workspaces, Terraform Cloud, Multi-Environment

Description: Learn how to configure and manage Terraform Cloud workspaces with the OpenTofu cloud backend for multi-environment deployments, including workspace naming, tagging, and variable sets.

## Introduction

Terraform Cloud workspaces organize infrastructure into logical units - one workspace per environment, region, or application component. The OpenTofu cloud backend's `workspaces` block selects which workspace to use, either by exact name or by tags. Proper workspace organization is the foundation of scalable multi-environment infrastructure management.

## Single Workspace Configuration

```hcl
# Single named workspace - simplest approach

terraform {
  cloud {
    organization = "my-company"

    workspaces {
      name = "production-us-east-1"
    }
  }
}
```

## Tag-Based Workspace Selection

```hcl
# Select workspace by tags - useful for multiple environments
terraform {
  cloud {
    organization = "my-company"

    workspaces {
      tags = ["infrastructure", "aws"]
    }
  }
}
```

```bash
# When using tags, init prompts for workspace selection
tofu init

# Or select via environment variable
export TF_WORKSPACE="production-us-east-1"
tofu init

# List available workspaces matching the tags
tofu workspace list
```

## Workspace Naming Conventions

```bash
# Common naming patterns:

# By environment + component
production-vpc
production-eks
production-rds
staging-vpc
staging-eks
development-vpc

# By region + environment
us-east-1-production
us-west-2-production
eu-west-1-production

# By team + environment
platform-production
platform-staging
application-production

# By account + region + environment (for multi-account setups)
account-123456-us-east-1-production
account-789012-eu-west-1-production
```

## Creating Workspaces at Scale

```bash
#!/bin/bash
# create-workspaces.sh

ORG="my-company"
ENVIRONMENTS=("development" "staging" "production")
COMPONENTS=("vpc" "eks" "rds" "alb")
REGIONS=("us-east-1" "eu-west-1")

create_workspace() {
  local NAME="$1"
  local TAGS="${2:-}"
  local EXEC_MODE="${3:-local}"

  curl -s -X POST \
    -H "Authorization: Bearer $TF_TOKEN" \
    -H "Content-Type: application/vnd.api+json" \
    "https://app.terraform.io/api/v2/organizations/${ORG}/workspaces" \
    -d "{
      \"data\": {
        \"type\": \"workspaces\",
        \"attributes\": {
          \"name\": \"$NAME\",
          \"execution-mode\": \"$EXEC_MODE\",
          \"auto-apply\": false,
          \"tag-names\": [${TAGS}]
        }
      }
    }"
}

for ENV in "${ENVIRONMENTS[@]}"; do
  for COMPONENT in "${COMPONENTS[@]}"; do
    NAME="${COMPONENT}-${ENV}"
    EXEC_MODE=$([[ "$ENV" == "production" ]] && echo "remote" || echo "local")
    TAGS="\"$ENV\", \"$COMPONENT\", \"aws\""

    echo "Creating workspace: $NAME (mode: $EXEC_MODE)"
    create_workspace "$NAME" "$TAGS" "$EXEC_MODE"
  done
done
```

## Variable Sets for Shared Configuration

```bash
# Create a variable set for AWS credentials shared across workspaces
curl -X POST \
  -H "Authorization: Bearer $TF_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/organizations/my-company/varsets" \
  -d '{
    "data": {
      "type": "varsets",
      "attributes": {
        "name": "AWS Production Credentials",
        "description": "AWS credentials for production account",
        "global": false
      },
      "relationships": {
        "vars": {
          "data": [
            {
              "type": "vars",
              "attributes": {
                "key": "AWS_ACCESS_KEY_ID",
                "value": "AKIA...",
                "category": "env",
                "sensitive": true
              }
            },
            {
              "type": "vars",
              "attributes": {
                "key": "AWS_SECRET_ACCESS_KEY",
                "value": "secret...",
                "category": "env",
                "sensitive": true
              }
            }
          ]
        }
      }
    }
  }'

# Assign variable set to workspaces with "production" tag
# (done via UI or API)
```

## Workspace Variables Configuration

```hcl
# Set workspace-specific Terraform variables
# These are terraform variables (not environment variables)

# workspace: vpc-production
# Variable: environment = "production"
# Variable: cidr = "10.0.0.0/16"

# workspace: vpc-staging
# Variable: environment = "staging"
# Variable: cidr = "10.1.0.0/16"

# In your configuration, read from variables normally:
variable "environment" {
  type = string
}

variable "cidr" {
  type = string
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  name   = "${var.environment}-vpc"
  cidr   = var.cidr
}
```

## Workspace Run Order and Dependencies

```bash
# Configure workspace run triggers
# Workspace B automatically queues a run when Workspace A completes

# Set up run trigger: eks-production triggers after vpc-production
VPC_WORKSPACE_ID="ws-vpc123"
EKS_WORKSPACE_ID="ws-eks456"

curl -X POST \
  -H "Authorization: Bearer $TF_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/run-triggers" \
  -d "{
    \"data\": {
      \"relationships\": {
        \"workspace\": {
          \"data\": {\"type\": \"workspaces\", \"id\": \"$EKS_WORKSPACE_ID\"}
        },
        \"sourceable\": {
          \"data\": {\"type\": \"workspaces\", \"id\": \"$VPC_WORKSPACE_ID\"}
        }
      }
    }
  }"
```

## Remote State Sharing Between Workspaces

```hcl
# Read outputs from another workspace's state
data "terraform_remote_state" "vpc" {
  backend = "remote"

  config = {
    organization = "my-company"
    workspaces = {
      name = "vpc-production"
    }
  }
}

resource "aws_eks_cluster" "main" {
  name = "production"

  vpc_config {
    # Use VPC outputs from the vpc-production workspace
    subnet_ids = data.terraform_remote_state.vpc.outputs.private_subnet_ids
  }
}
```

## Conclusion

Workspace organization in Terraform Cloud scales from single named workspaces for small teams to hundreds of workspaces for large organizations with multiple environments, regions, and components. Use consistent naming conventions to make workspace purpose obvious at a glance. Variable sets eliminate credential duplication across workspaces - define AWS credentials once in a variable set and assign it to all production workspaces. Run triggers create workspace dependency chains for ordered deployments (VPC before EKS, EKS before applications).
