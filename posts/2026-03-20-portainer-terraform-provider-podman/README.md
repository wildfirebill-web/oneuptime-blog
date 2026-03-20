# How to Use Portainer Terraform Provider with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Terraform, Podman, Infrastructure as Code, Automation, IaC

Description: Learn how to use the Portainer Terraform provider to manage Podman environments, stacks, and container configurations declaratively through Portainer's API.

---

The Portainer Terraform provider lets you manage Portainer resources (environments, stacks, registries, users) through Terraform's declarative configuration language. When Portainer is connected to a Podman socket, this extends to Podman environments.

## Setting Up the Portainer Terraform Provider

```hcl
# versions.tf
terraform {
  required_providers {
    portainer = {
      source  = "portainer/portainer"
      version = "~> 1.0"
    }
  }
}

provider "portainer" {
  endpoint = "http://localhost:9000"
  api_key  = var.portainer_api_key  # Create in Portainer > My Account > Access tokens
}
```

## Referencing a Podman Environment

In Portainer, add your Podman environment and note its ID. Then reference it in Terraform:

```hcl
# data.tf
data "portainer_endpoint" "podman_host" {
  name = "podman-server"  # The environment name you set in Portainer
}
```

## Deploying a Stack to the Podman Environment

```hcl
# stacks.tf
resource "portainer_stack" "webapp" {
  name          = "webapp"
  endpoint_id   = data.portainer_endpoint.podman_host.id
  type          = 2   # 2 = Standalone/Compose
  method        = "string"

  stack_file_content = <<-EOT
    version: "3.8"
    services:
      nginx:
        image: nginx:alpine
        ports:
          - "8080:80"
    EOT

  env {
    name  = "APP_ENV"
    value = "production"
  }
}
```

## Managing Registries via Terraform

```hcl
resource "portainer_registry" "internal" {
  name       = "Internal Registry"
  type       = 1   # 1 = Custom registry
  url        = "registry.example.com"
  username   = var.registry_username
  password   = var.registry_password
}
```

## Applying the Configuration

```bash
# Initialize Terraform
terraform init

# Set your API key
export TF_VAR_portainer_api_key="ptr_xxxxxxxxxxxx"

# Preview changes
terraform plan

# Apply
terraform apply
```

## Terraform State and Portainer

Terraform stores state about Portainer resources. When you `terraform destroy`, Portainer resources are removed. For stacks on Podman, this means the containers are stopped and removed.

Store Terraform state remotely (S3, Terraform Cloud) for team use:

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "portainer/terraform.tfstate"
    region = "us-east-1"
  }
}
```
