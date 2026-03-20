# How to Use Portainer Terraform Provider with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Terraform, Podman, Infrastructure as Code, Automation

Description: Use the Portainer Terraform provider to manage Podman-backed Portainer environments as code, enabling reproducible infrastructure deployments with stacks and settings defined in HCL.

## Introduction

The Portainer Terraform provider (`portainer/portainer`) allows you to manage Portainer resources - environments, stacks, users, teams, and settings - using Terraform HCL. When Portainer is connected to a Podman backend via the Docker-compatible socket, the same provider works seamlessly. This guide shows how to configure and use the Terraform provider against a Podman-backed Portainer instance.

## Prerequisites

- Portainer CE or BE running and connected to a Podman socket
- Terraform 1.0+ installed
- Portainer API access token or admin credentials
- Podman 4.0+ with socket enabled on the host

## Step 1: Install the Portainer Terraform Provider

```hcl
# versions.tf

terraform {
  required_version = ">= 1.0"

  required_providers {
    portainer = {
      source  = "portainer/portainer"
      version = "~> 1.0"
    }
  }
}
```

```bash
# Initialize Terraform - downloads the provider
terraform init
```

## Step 2: Configure the Provider

```hcl
# provider.tf
provider "portainer" {
  # Portainer server URL
  endpoint = "https://portainer.yourdomain.com:9443"

  # Authentication - use API key (recommended for automation)
  api_key = var.portainer_api_key

  # Or use username/password
  # username = "admin"
  # password = var.portainer_password

  # Skip TLS verification if using self-signed certs
  skip_tls_verify = true
}

variable "portainer_api_key" {
  description = "Portainer API key"
  sensitive   = true
}
```

```bash
# Generate an API key in Portainer:
# Profile → Access Tokens → Add Access Token
# Then set the variable:
export TF_VAR_portainer_api_key="ptr_xxxxxxxxxxxxxxxxxxxxxxxx"
```

## Step 3: Register the Podman Environment

```hcl
# environments.tf

# Register the Podman host as a Docker Standalone environment
# (Portainer sees it as Docker due to the compatible API)
resource "portainer_endpoint" "podman_host" {
  name           = "podman-production"
  endpoint_type  = 1  # Docker Standalone
  url            = "tcp://podman-host:2375"

  # Or use the socket directly if Portainer is on the same host
  # url = "unix:///run/podman/podman.sock"

  public_url     = "podman-host"

  # Group assignment
  group_id = portainer_endpoint_group.podman_group.id

  # Tag the environment
  tag_ids = [portainer_tag.production.id]
}

resource "portainer_endpoint_group" "podman_group" {
  name        = "Podman Hosts"
  description = "All Podman-backed environments"
}

resource "portainer_tag" "production" {
  name = "production"
}
```

## Step 4: Deploy a Stack via Terraform

```hcl
# stacks.tf

resource "portainer_stack" "webapp" {
  name         = "webapp"
  endpoint_id  = portainer_endpoint.podman_host.id
  stack_type   = 2  # Docker Compose (standalone)

  # Inline compose file
  stack_file_content = <<-YAML
    version: "3.8"
    services:
      web:
        image: nginx:alpine
        ports:
          - "8080:80"
        volumes:
          - web_data:/usr/share/nginx/html
        restart: unless-stopped
      db:
        image: postgres:16-alpine
        environment:
          POSTGRES_DB: webapp
          POSTGRES_USER: webuser
          POSTGRES_PASSWORD: ${var.db_password}
        volumes:
          - db_data:/var/lib/postgresql/data
        restart: unless-stopped
    volumes:
      web_data:
      db_data:
  YAML

  # Environment variables passed to the stack
  env {
    name  = "DB_HOST"
    value = "db"
  }
}

variable "db_password" {
  description = "Database password"
  sensitive   = true
}
```

## Step 5: Manage Users and Teams

```hcl
# users.tf

resource "portainer_user" "dev_user" {
  username = "developer1"
  password = var.dev_user_password
  role     = 2  # Standard user (1 = admin)
}

resource "portainer_team" "dev_team" {
  name = "developers"
}

resource "portainer_team_membership" "dev_membership" {
  user_id = portainer_user.dev_user.id
  team_id = portainer_team.dev_team.id
  role    = 1  # Leader (2 = member)
}

# Grant team access to the Podman environment
resource "portainer_endpoint_access" "podman_dev_access" {
  endpoint_id  = portainer_endpoint.podman_host.id
  team_id      = portainer_team.dev_team.id
  access_level = 2  # Read-Write
}

variable "dev_user_password" {
  sensitive = true
}
```

## Step 6: Configure Portainer Settings via Terraform

```hcl
# settings.tf

resource "portainer_settings" "global" {
  # Authentication settings
  authentication_method = 1  # Internal (2=LDAP, 3=OAuth)

  # Snapshot interval in seconds
  snapshot_interval = "300"

  # Edge agent settings
  enable_edge_compute_features = false

  # UI settings
  display_donation_header   = false
  display_external_contributors = false
}
```

## Step 7: Use Terraform State for Drift Detection

```bash
# Apply the configuration
terraform apply

# Check current state
terraform show

# Detect configuration drift
terraform plan  # Shows what would change

# Import existing Portainer resources
terraform import portainer_stack.existing_stack "endpoint_id:stack_id"

# Destroy all managed resources
terraform destroy
```

## Step 8: CI/CD Integration

```yaml
# .github/workflows/portainer-deploy.yml
name: Deploy to Portainer Podman

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        env:
          TF_VAR_portainer_api_key: ${{ secrets.PORTAINER_API_KEY }}
          TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        env:
          TF_VAR_portainer_api_key: ${{ secrets.PORTAINER_API_KEY }}
          TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}
        run: terraform apply tfplan
```

## Conclusion

The Portainer Terraform provider works identically whether Portainer is backed by Docker or Podman, since Portainer interacts with Podman through its Docker-compatible API. Using Terraform to manage Portainer enables version-controlled, reproducible infrastructure definitions for environments, stacks, users, and settings. For production Podman environments, combine Terraform state storage in a remote backend (S3, Terraform Cloud) with CI/CD pipelines to automate deployments whenever stack definitions change.
