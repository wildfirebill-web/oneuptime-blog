# How to Configure Azure App Service Deployment Slots with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, App Service, Deployment Slots, OpenTofu, Blue-Green, CI/CD

Description: Learn how to configure Azure App Service deployment slots with OpenTofu to enable zero-downtime blue-green deployments and staging environment validation.

## Overview

Azure App Service deployment slots allow you to deploy to a staging slot, validate the application, and then swap it into production with zero downtime. OpenTofu manages the slot configuration and swap settings.

## Step 1: Create the Production Web App

```hcl
# main.tf - Production web app
resource "azurerm_linux_web_app" "production" {
  name                = "my-app-production"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  service_plan_id     = azurerm_service_plan.plan.id
  https_only          = true

  site_config {
    application_stack {
      node_version = "20-lts"
    }
    always_on = true
  }

  # Sticky settings are NOT swapped between slots
  app_settings = {
    SLOT_NAME              = "production"
    DATABASE_URL           = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault_secret.prod_db_url.versionless_id})"
    APPLICATION_INSIGHTS_KEY = azurerm_application_insights.prod_insights.instrumentation_key
  }
}
```

## Step 2: Create a Staging Slot

```hcl
# Staging deployment slot for pre-production validation
resource "azurerm_linux_web_app_slot" "staging" {
  name           = "staging"
  app_service_id = azurerm_linux_web_app.production.id

  site_config {
    application_stack {
      node_version = "20-lts"
    }
    always_on = true
  }

  # Staging-specific settings
  app_settings = {
    SLOT_NAME    = "staging"
    DATABASE_URL = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault_secret.staging_db_url.versionless_id})"
  }
}
```

## Step 3: Configure Slot-Sticky Settings

```hcl
# Define which settings are "sticky" (not swapped with the slot)
resource "azurerm_web_app_active_slot" "swap_config" {
  # Note: In OpenTofu, we define the target production slot
  slot_id = azurerm_linux_web_app_slot.staging.id
}
```

## Step 4: Auto-Swap Configuration

```hcl
# Enable auto-swap: automatically swap staging to production after successful deploy
resource "azurerm_linux_web_app_slot" "staging_autoswap" {
  name           = "staging"
  app_service_id = azurerm_linux_web_app.production.id

  site_config {
    auto_swap_slot_name = "production"  # Swap to production automatically

    application_stack {
      node_version = "20-lts"
    }
  }

  app_settings = {
    SLOT_NAME = "staging"
  }
}
```

## Step 5: Multiple Slots for Different Environments

```hcl
# Create slots for different pre-production environments
locals {
  slots = ["staging", "canary", "hotfix"]
}

resource "azurerm_linux_web_app_slot" "slots" {
  for_each = toset(local.slots)

  name           = each.value
  app_service_id = azurerm_linux_web_app.production.id

  site_config {
    application_stack {
      node_version = "20-lts"
    }
  }

  app_settings = {
    SLOT_NAME    = each.value
    ENVIRONMENT  = "pre-production"
  }
}
```

## Step 6: Outputs

```hcl
output "production_url" {
  value = "https://${azurerm_linux_web_app.production.default_hostname}"
}

output "staging_url" {
  value = "https://${azurerm_linux_web_app_slot.staging.default_hostname}"
}
```

## Summary

Azure App Service deployment slots with OpenTofu enable blue-green deployments with zero downtime. The staging slot receives new code, gets validated, and then swaps with production. Sticky settings ensure production secrets and configuration stay with the production slot throughout the swap.
