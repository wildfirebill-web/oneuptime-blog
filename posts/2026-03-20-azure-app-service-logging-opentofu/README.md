# How to Configure Azure App Service Logging with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, App Service, Logging, OpenTofu, Monitoring, Application Insights

Description: Learn how to configure Azure App Service application logging, HTTP logging, and diagnostic settings with OpenTofu for comprehensive observability.

## Overview

Azure App Service supports multiple logging types: application logs, HTTP access logs, detailed error pages, and failed request tracing. OpenTofu configures these alongside Application Insights for centralized observability.

## Step 1: Create Log Analytics and Application Insights

```hcl
# main.tf - Observability resources
resource "azurerm_log_analytics_workspace" "law" {
  name                = "app-logging-workspace"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

resource "azurerm_application_insights" "insights" {
  name                = "my-app-insights"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  workspace_id        = azurerm_log_analytics_workspace.law.id
  application_type    = "web"
}
```

## Step 2: Create Storage for Logs

```hcl
# Storage account for HTTP logs and error traces
resource "azurerm_storage_account" "log_storage" {
  name                     = "applogstorage"
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_storage_container" "http_logs" {
  name                  = "http-logs"
  storage_account_name  = azurerm_storage_account.log_storage.name
  container_access_type = "private"
}
```

## Step 3: Configure App Service Logs

```hcl
resource "azurerm_linux_web_app" "app" {
  name                = "my-logged-app"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  service_plan_id     = azurerm_service_plan.plan.id

  # Application Insights auto-instrumentation
  app_settings = {
    APPLICATIONINSIGHTS_CONNECTION_STRING = azurerm_application_insights.insights.connection_string
    ApplicationInsightsAgent_EXTENSION_VERSION = "~3"
  }

  logs {
    # Application logs (stdout/stderr)
    application_logs {
      file_system_level = "Warning"  # Verbose, Information, Warning, Error, Off

      # Stream to Azure Blob Storage for long-term retention
      azure_blob_storage {
        level             = "Information"
        sas_url           = "${azurerm_storage_account.log_storage.primary_blob_endpoint}${azurerm_storage_container.http_logs.name}${data.azurerm_storage_account_sas.log_sas.sas}"
        retention_in_days = 30
      }
    }

    # HTTP access logs
    http_logs {
      file_system {
        retention_in_days = 7
        retention_in_mb   = 100
      }
    }

    # Enable detailed error messages
    detailed_error_messages = true

    # Enable failed request tracing
    failed_request_tracing = true
  }

  site_config {}
}
```

## Step 4: Diagnostic Settings to Log Analytics

```hcl
# Stream App Service logs to Log Analytics for querying
resource "azurerm_monitor_diagnostic_setting" "app_diagnostics" {
  name               = "app-service-diagnostics"
  target_resource_id = azurerm_linux_web_app.app.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.law.id

  enabled_log {
    category = "AppServiceHTTPLogs"
  }

  enabled_log {
    category = "AppServiceConsoleLogs"
  }

  enabled_log {
    category = "AppServiceAppLogs"
  }

  enabled_log {
    category = "AppServiceAuditLogs"
  }

  metric {
    category = "AllMetrics"
    enabled  = true
  }
}
```

## Step 5: Outputs

```hcl
output "application_insights_connection_string" {
  value     = azurerm_application_insights.insights.connection_string
  sensitive = true
}

output "log_analytics_workspace_id" {
  value = azurerm_log_analytics_workspace.law.workspace_id
}
```

## Summary

Azure App Service logging configured with OpenTofu provides comprehensive observability through multiple layers: file system logs for quick debugging, Application Insights for distributed tracing, and Log Analytics for long-term log querying. Enabling all three ensures no visibility gaps in production applications.
