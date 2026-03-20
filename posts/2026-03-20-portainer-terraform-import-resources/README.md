# How to Import Existing Portainer Resources into Terraform - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Terraform, Infrastructure, Migration, DevOps

Description: Learn how to import existing Portainer resources - environments, users, teams, and stacks - into Terraform state so you can manage them as code without recreating them.

## Introduction

When you start using Terraform with an existing Portainer installation, you need to import your current resources into Terraform state. Without importing, Terraform will try to recreate resources that already exist, causing conflicts. This guide covers how to discover and import all types of Portainer resources.

## Prerequisites

- Portainer Terraform provider configured
- Existing Portainer resources to import
- Portainer API access token
- `curl` and `jq` for resource discovery

## Understanding Terraform Import

The `terraform import` command associates an existing resource with a Terraform resource definition:

```bash
# General syntax

terraform import <resource_type>.<resource_name> <resource_id>
```

After importing, Terraform knows about the resource and will manage it going forward. You still need to write the matching Terraform configuration.

## Step 1: Discover Existing Resource IDs

Before importing, find the IDs of your existing resources via the Portainer API:

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-api-key"

echo "=== Portainer Resource Discovery ==="

# List environments
echo ""
echo "Environments:"
curl -s -H "X-API-Key: $TOKEN" "$PORTAINER_URL/api/endpoints" | \
  jq -r '.[] | "  ID: \(.Id)  Name: \(.Name)  Type: \(.Type)"'

# List users
echo ""
echo "Users:"
curl -s -H "X-API-Key: $TOKEN" "$PORTAINER_URL/api/users" | \
  jq -r '.[] | "  ID: \(.Id)  Username: \(.Username)  Role: \(.Role)"'

# List teams
echo ""
echo "Teams:"
curl -s -H "X-API-Key: $TOKEN" "$PORTAINER_URL/api/teams" | \
  jq -r '.[] | "  ID: \(.Id)  Name: \(.Name)"'

# List stacks
echo ""
echo "Stacks:"
curl -s -H "X-API-Key: $TOKEN" "$PORTAINER_URL/api/stacks" | \
  jq -r '.[] | "  ID: \(.Id)  Name: \(.Name)  Endpoint: \(.EndpointId)"'

# List registries
echo ""
echo "Registries:"
curl -s -H "X-API-Key: $TOKEN" "$PORTAINER_URL/api/registries" | \
  jq -r '.[] | "  ID: \(.Id)  Name: \(.Name)  URL: \(.URL)"'
```

## Step 2: Write Terraform Configuration for Existing Resources

Before importing, write the Terraform config that matches your existing resources:

```hcl
# environments.tf - Match your existing environment
resource "portainer_environment" "production" {
  name             = "production"         # Must match actual name
  environment_url  = "unix:///var/run/docker.sock"
  environment_type = 1
}

# users.tf - Match your existing users
resource "portainer_user" "admin_user" {
  username = "john.doe"
  password = "placeholder"  # Required by provider but won't be changed on import
  role     = 2
}

# teams.tf - Match your existing teams
resource "portainer_team" "devops" {
  name = "devops"
}

# registries.tf - Match existing registry
resource "portainer_registry" "harbor" {
  name           = "Company Harbor"
  registry_type  = 6
  url            = "registry.company.com"
  authentication = true
  username       = "portainer-svc"
  password       = "placeholder"  # Will not overwrite on import
}

# stacks.tf - Match existing stack
resource "portainer_stack" "myapp" {
  name               = "my-app"
  endpoint_id        = 1
  stack_file_content = file("stacks/myapp/docker-compose.yml")
}
```

## Step 3: Import Each Resource

```bash
# Import environment (use the ID from Step 1)
terraform import portainer_environment.production 1

# Import a user
terraform import portainer_user.admin_user 3

# Import a team
terraform import portainer_team.devops 2

# Import a registry
terraform import portainer_registry.harbor 4

# Import a stack
terraform import portainer_stack.myapp 5
```

## Step 4: Verify the Import

After importing, verify Terraform's state matches reality:

```bash
# View the imported state
terraform state show portainer_environment.production
terraform state show portainer_user.admin_user

# Check what Terraform wants to change (should be minimal or empty)
terraform plan
```

If `terraform plan` shows many changes, your Terraform config doesn't match the actual resource state. Adjust your config to match.

## Step 5: Bulk Import Script

For many resources, use a script:

```bash
#!/bin/bash
# bulk-import.sh - Import all Portainer resources

PORTAINER_URL="https://portainer.example.com"
TOKEN="your-api-key"

# Import all environments
curl -s -H "X-API-Key: $TOKEN" "$PORTAINER_URL/api/endpoints" | \
  jq -r '.[] | "terraform import portainer_environment.\(.Name | gsub("[^a-zA-Z0-9_]"; "_")) \(.Id)"' | \
  while read -r CMD; do
    echo "Running: $CMD"
    eval "$CMD" || echo "  FAILED"
  done

# Import all users
curl -s -H "X-API-Key: $TOKEN" "$PORTAINER_URL/api/users" | \
  jq -r '.[] | "terraform import portainer_user.\(.Username | gsub("[^a-zA-Z0-9_]"; "_")) \(.Id)"' | \
  while read -r CMD; do
    echo "Running: $CMD"
    eval "$CMD" || echo "  FAILED"
  done

# Import all teams
curl -s -H "X-API-Key: $TOKEN" "$PORTAINER_URL/api/teams" | \
  jq -r '.[] | "terraform import portainer_team.\(.Name | gsub("[^a-zA-Z0-9_]"; "_")) \(.Id)"' | \
  while read -r CMD; do
    echo "Running: $CMD"
    eval "$CMD" || echo "  FAILED"
  done

echo "Import complete. Run 'terraform plan' to check for drift."
```

## Step 6: Using terraform import blocks (Terraform 1.5+)

Terraform 1.5+ supports `import` blocks in configuration:

```hcl
# imports.tf - Terraform 1.5+ import blocks
import {
  to = portainer_environment.production
  id = "1"
}

import {
  to = portainer_user.admin_user
  id = "3"
}

import {
  to = portainer_team.devops
  id = "2"
}

import {
  to = portainer_stack.myapp
  id = "5"
}
```

Then run:

```bash
# Generate config from import (Terraform 1.5+)
terraform plan -generate-config-out=generated.tf

# Review generated.tf and clean it up
# Then apply
terraform apply
```

## Step 7: Handle Sensitive Fields

Some resources have sensitive fields (passwords) that can't be exported from Portainer's API. Handle these carefully:

```hcl
# Use variables for sensitive imported resources
variable "imported_user_passwords" {
  description = "Passwords for imported users (not managed by Terraform)"
  type        = map(string)
  sensitive   = true
  default     = {}
}

resource "portainer_user" "imported" {
  username = "existing-user"
  # Use a placeholder if Terraform manages the resource but not the password
  password = lookup(var.imported_user_passwords, "existing-user", "IMPORTED-DO-NOT-CHANGE")
  role     = 2

  lifecycle {
    ignore_changes = [password]  # Don't manage password changes
  }
}
```

## Conclusion

Importing existing Portainer resources into Terraform state is the essential first step when adopting infrastructure-as-code for an established Portainer installation. Discover your resource IDs via the API, write matching Terraform configuration, run `terraform import` for each resource, and verify with `terraform plan` that the desired state matches reality. This process gives you full Terraform control without disrupting existing deployments.
