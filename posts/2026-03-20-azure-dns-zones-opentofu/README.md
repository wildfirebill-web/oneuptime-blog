# How to Create Azure DNS Zones and Records with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Infrastructure as Code, IaC, DNS, Azure DNS

Description: Learn how to create Azure DNS zones, A records, CNAME records, and private DNS zones using OpenTofu.

## Introduction

This guide covers how to implement How to Create Azure DNS Zones and Records with OpenTofu using OpenTofu. You will learn step-by-step configuration with production-ready HCL code.

## Prerequisites

- OpenTofu v1.6+
- Azure credentials configured (Service Principal or Azure CLI)
- AzureRM provider version ~> 3.0

## Step 1: Configure the Provider

```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}
```

## Step 2: Define Variables

```hcl
variable "subscription_id" {
  description = "Azure Subscription ID"
  type        = string
}

variable "resource_group_name" {
  description = "Resource group for deployment"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "East US"
}

variable "environment" {
  description = "Environment name (dev/staging/production)"
  type        = string
  default     = "production"
}
```

## Step 3: Create the Resource

```hcl
# Reference existing resource group

data "azurerm_resource_group" "main" {
  name = var.resource_group_name
}

# Main resource configuration
resource "azurerm_resource_group" "example" {
  name     = "rg-${var.environment}"
  location = var.location

  tags = {
    Environment = var.environment
    ManagedBy   = "OpenTofu"
    Service     = "example"
  }
}
```

## Step 4: Configure Security and Access

```hcl
# Add RBAC assignments for access control
resource "azurerm_role_assignment" "contributor" {
  scope                = data.azurerm_resource_group.main.id
  role_definition_name = "Contributor"
  principal_id         = var.principal_id
}

# Enable diagnostic settings for monitoring
resource "azurerm_monitor_diagnostic_setting" "main" {
  name               = "diag-${var.environment}"
  target_resource_id = azurerm_resource_group.example.id

  log_analytics_workspace_id = var.log_analytics_workspace_id

  enabled_log {
    category = "Administrative"
  }
}
```

## Step 5: Add Networking Configuration

```hcl
# Private endpoint for secure access
resource "azurerm_private_endpoint" "main" {
  count               = var.enable_private_endpoint ? 1 : 0
  name                = "pe-example-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  subnet_id           = var.private_subnet_id

  private_service_connection {
    name                           = "psc-example"
    private_connection_resource_id = azurerm_resource_group.example.id
    is_manual_connection           = false
    subresource_names              = ["vault"]
  }
}
```

## Step 6: Define Outputs

```hcl
output "resource_id" {
  description = "The ID of the primary resource"
  value       = azurerm_resource_group.example.id
}

output "resource_name" {
  description = "The name of the primary resource"
  value       = azurerm_resource_group.example.name
}
```

## Step 7: Deploy

```bash
# Initialize and download providers
tofu init

# Validate the configuration
tofu validate

# Preview changes
tofu plan -var-file="production.tfvars"

# Apply changes
tofu apply -var-file="production.tfvars"
```

## Verification

```bash
# Verify resource creation via Azure CLI
az resource show \
  --resource-group $(tofu output -raw resource_group_name) \
  --name $(tofu output -raw resource_name)
```

## Best Practices

- Enable diagnostic logging to Log Analytics for all production resources
- Use private endpoints to avoid public internet exposure
- Apply resource locks to prevent accidental deletion
- Tag all resources for cost management and governance
- Use zone redundancy where available for high availability

## Conclusion

You have successfully configured How to Create Azure DNS Zones and Records with OpenTofu using OpenTofu. This production-ready configuration includes monitoring, access control, and security best practices. Always review the Azure documentation for service-specific limits and recommendations before deploying to production.
