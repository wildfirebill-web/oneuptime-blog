# How to Configure Mysql Provider with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Provider, Automation, DevOps

Description: Learn how to configure and use the Mysql provider in OpenTofu to manage Mysql resources as code.

## Introduction

The Mysql provider for OpenTofu enables managing Mysql resources with the same plan/apply workflow as your cloud infrastructure. This guide covers authentication, basic resource configuration, and production best practices.

## Provider Installation

```hcl
terraform {
  required_providers {
    # Replace with the actual provider source
    provider_name = {
      source  = "provider-namespace/provider-name"
      version = "~> 1.0"
    }
  }
  required_version = ">= 1.6.0"
}
```

## Authentication

Most providers read credentials from environment variables:

```bash
# Set provider credentials via environment variables

export PROVIDER_API_KEY="your-api-key"
export PROVIDER_API_SECRET="your-api-secret"
```

```hcl
provider "provider_name" {
  # Credentials are read from environment variables
  # api_key = var.api_key  # Alternative: inline (not recommended)
}
```

## Example Resource

```hcl
# Example resource demonstrating provider usage
resource "provider_example_resource" "main" {
  name = "${var.name}-${var.environment}"

  tags = {
    environment = var.environment
    managed_by  = "opentofu"
  }
}
```

## Variables

```hcl
variable "name"        { type = string }
variable "environment" { type = string }
```

## Outputs

```hcl
output "resource_id" { value = provider_example_resource.main.id }
```

## Best Practices

- Store API keys in environment variables or a secrets manager-never in .tf files
- Pin provider versions in `required_providers` to prevent unexpected updates
- Commit the `.terraform.lock.hcl` file to lock exact provider versions
- Use separate provider configurations per environment using aliases or workspaces

## Conclusion

Managing Mysql resources with OpenTofu brings the same consistency and auditability to SaaS tooling as you get with cloud infrastructure. Start by codifying your most critical resources and gradually expand coverage over time.
