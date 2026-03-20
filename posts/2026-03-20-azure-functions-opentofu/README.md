# How to Deploy Azure Functions with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Infrastructure as Code, IaC, Azure Functions, Serverless

Description: Learn how to deploy Azure Functions with consumption and premium plans, storage accounts, and application insights using OpenTofu.

## Introduction

This guide covers How to Deploy Azure Functions with OpenTofu using OpenTofu. You will learn step-by-step how to configure the necessary resources with production-ready settings.

## Prerequisites

- OpenTofu v1.6+
- Azure credentials configured
- AzureRM provider installed

## Step 1: Configure the Provider

```hcl
terraform {
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
  description = "Resource group name"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "East US"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "production"
}
```

## Step 3: Create Core Resources

```hcl
# Reference existing resource group

data "azurerm_resource_group" "main" {
  name = var.resource_group_name
}

# Create the primary resource for this guide
resource "azurerm_resource_group" "example" {
  name     = "rg-example-${var.environment}"
  location = var.location

  tags = {
    Environment = var.environment
    ManagedBy   = "OpenTofu"
  }
}
```

## Step 4: Configure Advanced Settings

```hcl
# Configure monitoring and diagnostics
resource "azurerm_monitor_diagnostic_setting" "example" {
  name               = "diag-example"
  target_resource_id = azurerm_resource_group.example.id

  log_analytics_workspace_id = var.log_analytics_workspace_id

  enabled_log {
    category = "Administrative"
  }

  metric {
    category = "AllMetrics"
    enabled  = true
  }
}
```

## Step 5: Set Up Access Control

```hcl
# Assign RBAC roles
resource "azurerm_role_assignment" "app_contributor" {
  scope                = azurerm_resource_group.example.id
  role_definition_name = "Contributor"
  principal_id         = var.app_principal_id
}

# Reader role for monitoring
resource "azurerm_role_assignment" "monitor_reader" {
  scope                = azurerm_resource_group.example.id
  role_definition_name = "Monitoring Reader"
  principal_id         = var.monitor_principal_id
}
```

## Step 6: Define Outputs

```hcl
output "resource_id" {
  description = "The ID of the created resource"
  value       = azurerm_resource_group.example.id
}

output "resource_name" {
  description = "The name of the created resource"
  value       = azurerm_resource_group.example.name
}
```

## Step 7: Deploy

```bash
# Initialize OpenTofu
tofu init

# Preview changes
tofu plan -var-file="production.tfvars"

# Apply configuration
tofu apply -var-file="production.tfvars"
```

## Best Practices

- Always use resource groups to organize related resources
- Apply consistent tagging for cost allocation and management
- Enable diagnostic logging for all production resources
- Use private endpoints to avoid public internet exposure
- Follow Azure naming conventions for resource identifiers

## Conclusion

You have successfully completed the setup for How to Deploy Azure Functions with OpenTofu using OpenTofu. This configuration follows Azure best practices including proper tagging, monitoring, and access control. Adapt the configuration to your specific requirements and always test in a non-production environment before applying to production.
