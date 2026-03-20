# How to Import Azure Resource Groups into OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Azure, Resource Groups, Import, ARM

Description: Learn how to import existing Azure Resource Groups and their resources into OpenTofu state using Azure resource IDs and declarative import blocks.

## Introduction

Azure Resource Groups are the container for all Azure resources. Importing them into OpenTofu is usually the first step when bringing an existing Azure environment under infrastructure-as-code management.

## Step 1: Find Resource Group Details

```bash
RESOURCE_GROUP="my-app-rg"

# Get resource group details

az group show --name $RESOURCE_GROUP --output json | jq '{
  name: .name,
  location: .location,
  tags: .tags
}'

# List all resources in the group
az resource list --resource-group $RESOURCE_GROUP \
  --query '[].{name:name,type:type,id:id}' \
  --output table

# Get the resource group's ARM ID (needed for import)
az group show --name $RESOURCE_GROUP --query id --output tsv
# Output: /subscriptions/12345678-1234-5678-9012-123456789012/resourceGroups/my-app-rg
```

## Step 2: Write Matching HCL

```hcl
resource "azurerm_resource_group" "app" {
  name     = "my-app-rg"
  location = "East US"  # Must match existing location

  tags = {
    Environment = "prod"
    Project     = "myapp"
    ManagedBy   = "OpenTofu"
  }
}
```

## Step 3: Import the Resource Group

```hcl
# import.tf - Using the full ARM resource ID
import {
  to = azurerm_resource_group.app
  id = "/subscriptions/12345678-1234-5678-9012-123456789012/resourceGroups/my-app-rg"
}
```

```bash
# Or use the CLI import command
tofu import azurerm_resource_group.app \
  "/subscriptions/12345678-1234-5678-9012-123456789012/resourceGroups/my-app-rg"
```

## Importing Multiple Resource Groups

```hcl
variable "resource_groups" {
  type = map(object({
    name     = string
    location = string
    tags     = map(string)
  }))
}

resource "azurerm_resource_group" "groups" {
  for_each = var.resource_groups

  name     = each.value.name
  location = each.value.location
  tags     = each.value.tags
}

# Import each resource group
import {
  to = azurerm_resource_group.groups["app"]
  id = "/subscriptions/${var.subscription_id}/resourceGroups/my-app-rg"
}

import {
  to = azurerm_resource_group.groups["data"]
  id = "/subscriptions/${var.subscription_id}/resourceGroups/my-data-rg"
}
```

## Finding ARM IDs for Import

```bash
# The ARM ID format for resources is standardized:
# /subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{provider}/{type}/{name}

# Get subscription ID
az account show --query id --output tsv

# Find ARM IDs for multiple resources
az resource list --resource-group my-app-rg \
  --query '[].{Name:name,Type:type,ID:id}' \
  --output table
```

## Importing Azure Lock on Resource Group

```hcl
resource "azurerm_management_lock" "app_rg_lock" {
  name       = "protect-prod-rg"
  scope      = azurerm_resource_group.app.id
  lock_level = "CanNotDelete"
  notes      = "This resource group contains production infrastructure"
}

import {
  to = azurerm_management_lock.app_rg_lock
  id = "${azurerm_resource_group.app.id}/providers/Microsoft.Authorization/locks/protect-prod-rg"
}
```

## Conclusion

Azure Resource Group import is usually the entry point for bringing existing Azure infrastructure under OpenTofu management. The full ARM resource ID (including subscription ID) is required for import. After importing the resource group, continue importing its contents by listing them with `az resource list` and importing each in dependency order.
