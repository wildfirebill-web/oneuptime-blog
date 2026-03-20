# How to Set Up Azure Storage Replication with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, Storage, OpenTofu, Replication, Disaster Recovery, Infrastructure

Description: Learn how to configure Azure Storage replication options including LRS, ZRS, GRS, and GZRS with OpenTofu for durability and disaster recovery.

## Overview

Azure Storage offers multiple replication strategies to protect your data. OpenTofu makes it straightforward to configure and switch between replication types as part of your infrastructure code.

## Replication Types

| Type | Copies | Scope | Use Case |
|------|--------|-------|----------|
| LRS  | 3      | Single datacenter | Dev/test, non-critical data |
| ZRS  | 3      | Availability zones | High availability within region |
| GRS  | 6      | Two regions | Disaster recovery |
| GZRS | 6      | Zones + two regions | Maximum durability |

## Step 1: LRS and ZRS Configuration

```hcl
# variables.tf

variable "replication_type" {
  description = "Storage replication type: LRS, ZRS, GRS, RAGRS, GZRS, RAGZRS"
  type        = string
  default     = "ZRS"

  validation {
    condition     = contains(["LRS", "ZRS", "GRS", "RAGRS", "GZRS", "RAGZRS"], var.replication_type)
    error_message = "Replication type must be one of: LRS, ZRS, GRS, RAGRS, GZRS, RAGZRS."
  }
}
```

```hcl
# main.tf - Zone-redundant storage for high availability
resource "azurerm_storage_account" "zrs_storage" {
  name                     = "myzrsstorage"
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "ZRS"  # Replicated across 3 availability zones
}
```

## Step 2: Geo-Redundant Storage (GRS)

```hcl
# GRS replicates data to a secondary region automatically
resource "azurerm_storage_account" "grs_storage" {
  name                     = "mygrsstorage"
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "RAGRS"  # Read-Access GRS enables reads from secondary

  # Enable secondary endpoint access for read operations during regional outages
  cross_tenant_replication_enabled = false
}

output "primary_blob_endpoint" {
  value = azurerm_storage_account.grs_storage.primary_blob_endpoint
}

output "secondary_blob_endpoint" {
  value       = azurerm_storage_account.grs_storage.secondary_blob_endpoint
  description = "Secondary endpoint available with RAGRS for read-only access"
}
```

## Step 3: Geo-Zone-Redundant Storage (GZRS)

```hcl
# GZRS combines zone redundancy with geo-replication for maximum protection
resource "azurerm_storage_account" "gzrs_storage" {
  name                     = "mygzrsstorage"
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "RAGZRS"  # Read-Access GZRS
}
```

## Step 4: Object Replication Between Accounts

For granular replication between specific containers:

```hcl
# Source storage account (must have versioning and change feed enabled)
resource "azurerm_storage_account" "source" {
  name                     = "mysourcestorage"
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = "eastus"
  account_tier             = "Standard"
  account_replication_type = "LRS"

  blob_properties {
    versioning_enabled  = true
    change_feed_enabled = true
  }
}

# Destination storage account
resource "azurerm_storage_account" "destination" {
  name                     = "mydeststorage"
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = "westus"
  account_tier             = "Standard"
  account_replication_type = "LRS"

  blob_properties {
    versioning_enabled = true
  }
}

# Object replication policy
resource "azurerm_storage_object_replication" "replication" {
  source_storage_account_id      = azurerm_storage_account.source.id
  destination_storage_account_id = azurerm_storage_account.destination.id

  rules {
    source_container_name      = azurerm_storage_container.src_container.name
    destination_container_name = azurerm_storage_container.dst_container.name
    # Replicate only blobs with this prefix
    filter_out_blobs_with_prefix = ["important/"]
  }
}
```

## Summary

Choosing the right Azure Storage replication strategy with OpenTofu depends on your RPO/RTO requirements and budget. Use ZRS for zone-level HA, RAGRS/RAGZRS for cross-region DR with read access, and object replication for fine-grained container-level replication policies.
