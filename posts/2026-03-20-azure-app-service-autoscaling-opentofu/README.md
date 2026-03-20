# How to Set Up Azure App Service Autoscaling with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, App Service, Autoscaling, OpenTofu, Performance, Infrastructure

Description: Learn how to configure Azure App Service autoscaling with OpenTofu using metric-based rules, scheduled scaling, and scale-in/out policies for handling variable workloads.

## Overview

Azure App Service autoscaling automatically adjusts the number of instances based on demand. OpenTofu configures autoscale settings with metric-based rules, scheduled profiles, and notification actions.

## Step 1: Create App Service Plan and Web App

```hcl
# main.tf - Standard or Premium plan is required for autoscaling
resource "azurerm_service_plan" "autoscale_plan" {
  name                = "autoscale-app-plan"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  os_type             = "Linux"
  sku_name            = "S2"  # Standard S2 - minimum for autoscale

  # Initial instance count
  worker_count = 2
}
```

## Step 2: Configure Autoscale Settings

```hcl
# Autoscale settings attached to the App Service Plan
resource "azurerm_monitor_autoscale_setting" "app_autoscale" {
  name                = "app-autoscale"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  target_resource_id  = azurerm_service_plan.autoscale_plan.id
  enabled             = true

  # Default profile - metric-based scaling
  profile {
    name = "defaultProfile"

    capacity {
      default = 2   # Default instance count
      minimum = 1   # Scale in to minimum 1 instance
      maximum = 10  # Scale out to maximum 10 instances
    }

    # Scale OUT when CPU > 70% for 10 minutes
    rule {
      metric_trigger {
        metric_name        = "CpuPercentage"
        metric_resource_id = azurerm_service_plan.autoscale_plan.id
        operator           = "GreaterThan"
        statistic          = "Average"
        threshold          = 70
        time_aggregation   = "Average"
        time_grain         = "PT1M"
        time_window        = "PT10M"
      }

      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = 2  # Add 2 instances at a time
        cooldown  = "PT5M"  # Wait 5 minutes before scaling again
      }
    }

    # Scale IN when CPU < 30% for 10 minutes
    rule {
      metric_trigger {
        metric_name        = "CpuPercentage"
        metric_resource_id = azurerm_service_plan.autoscale_plan.id
        operator           = "LessThan"
        statistic          = "Average"
        threshold          = 30
        time_aggregation   = "Average"
        time_grain         = "PT1M"
        time_window        = "PT10M"
      }

      scale_action {
        direction = "Decrease"
        type      = "ChangeCount"
        value     = 1  # Remove 1 instance at a time
        cooldown  = "PT10M"
      }
    }
  }

  # Scheduled profile for known traffic peaks (e.g., business hours)
  profile {
    name = "businessHours"

    capacity {
      default = 4
      minimum = 4
      maximum = 10
    }

    recurrence {
      timezone = "UTC"
      days     = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"]
      hours    = [8]    # Start at 8 AM UTC
      minutes  = [0]
    }
  }

  # Notification when scaling events occur
  notification {
    email {
      send_to_subscription_administrator    = true
      send_to_subscription_co_administrator = false
      custom_emails                          = ["ops-team@example.com"]
    }
  }
}
```

## Step 3: Memory-Based Scaling Rule

```hcl
# Add memory pressure scaling rule to the autoscale setting
resource "azurerm_monitor_autoscale_setting" "memory_autoscale" {
  name                = "memory-autoscale"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  target_resource_id  = azurerm_service_plan.autoscale_plan.id

  profile {
    name = "memoryProfile"

    capacity {
      default = 2
      minimum = 2
      maximum = 8
    }

    rule {
      metric_trigger {
        metric_name        = "MemoryPercentage"
        metric_resource_id = azurerm_service_plan.autoscale_plan.id
        operator           = "GreaterThan"
        statistic          = "Average"
        threshold          = 80
        time_aggregation   = "Average"
        time_grain         = "PT1M"
        time_window        = "PT5M"
      }

      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = 1
        cooldown  = "PT5M"
      }
    }
  }
}
```

## Summary

Azure App Service autoscaling configured with OpenTofu dynamically adjusts your application's compute capacity. Combining metric-based rules for CPU and memory with scheduled profiles for known traffic peaks ensures cost-efficient scaling that handles both predictable and unpredictable load changes.
