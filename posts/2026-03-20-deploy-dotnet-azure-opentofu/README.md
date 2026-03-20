# How to Deploy a .NET Application with OpenTofu on Azure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, .NET, Azure, App Service, Infrastructure as Code

Description: Learn how to deploy a .NET 8 application on Azure using OpenTofu, with Azure App Service for hosting, Azure SQL for the database, and Azure Key Vault for secrets.

## Introduction

.NET applications deploy naturally on Azure using App Service (managed PaaS) or Azure Container Apps (containerized). This guide uses Azure App Service with deployment slots for zero-downtime deployments, Azure SQL Database, and Azure Key Vault for secret management.

## Resource Group and App Service Plan

```hcl
resource "azurerm_resource_group" "app" {
  name     = "myapp-${var.environment}-rg"
  location = var.location
}

resource "azurerm_service_plan" "app" {
  name                = "myapp-${var.environment}-plan"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location
  os_type             = "Linux"
  sku_name            = var.environment == "prod" ? "P2v3" : "B2"
}
```

## Azure SQL Database

```hcl
resource "azurerm_mssql_server" "app" {
  name                         = "myapp-${var.environment}-sql"
  resource_group_name          = azurerm_resource_group.app.name
  location                     = azurerm_resource_group.app.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.sql_admin_password

  azuread_administrator {
    login_username              = "AzureAD Admin"
    object_id                   = var.admin_object_id
    azuread_authentication_only = false
  }
}

resource "azurerm_mssql_database" "app" {
  name           = "myapp"
  server_id      = azurerm_mssql_server.app.id
  collation      = "SQL_Latin1_General_CP1_CI_AS"
  sku_name       = var.environment == "prod" ? "S2" : "Basic"
  zone_redundant = var.environment == "prod"

  lifecycle {
    prevent_destroy = true
  }
}
```

## Azure Key Vault for Secrets

```hcl
data "azurerm_client_config" "current" {}

resource "azurerm_key_vault" "app" {
  name                = "myapp-${var.environment}-kv"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"

  enable_rbac_authorization = true
  purge_protection_enabled  = var.environment == "prod"
}

resource "azurerm_key_vault_secret" "db_connection" {
  name         = "ConnectionStrings--DefaultConnection"
  value        = "Server=${azurerm_mssql_server.app.fully_qualified_domain_name};Database=myapp;User Id=sqladmin;Password=${var.sql_admin_password};Encrypt=True"
  key_vault_id = azurerm_key_vault.app.id
}
```

## .NET App Service with Managed Identity

```hcl
resource "azurerm_linux_web_app" "dotnet" {
  name                = "myapp-${var.environment}-web"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location
  service_plan_id     = azurerm_service_plan.app.id

  identity {
    type = "SystemAssigned"  # enables Key Vault access
  }

  site_config {
    application_stack {
      dotnet_version = "8.0"
    }

    always_on = var.environment == "prod"

    health_check_path = "/health"
  }

  app_settings = {
    ASPNETCORE_ENVIRONMENT = var.environment == "prod" ? "Production" : "Staging"
    APPLICATIONINSIGHTS_CONNECTION_STRING = azurerm_application_insights.app.connection_string

    # Reference Key Vault secrets
    "ConnectionStrings__DefaultConnection" = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault_secret.db_connection.id})"
  }

  connection_string {
    name  = "DefaultConnection"
    type  = "SQLServer"
    value = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault_secret.db_connection.id})"
  }
}

# Grant App Service access to Key Vault
resource "azurerm_role_assignment" "app_keyvault" {
  scope                = azurerm_key_vault.app.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_linux_web_app.dotnet.identity[0].principal_id
}
```

## Deployment Slots for Zero-Downtime Deploys

```hcl
resource "azurerm_linux_web_app_slot" "staging" {
  name           = "staging"
  app_service_id = azurerm_linux_web_app.dotnet.id

  site_config {
    application_stack {
      dotnet_version = "8.0"
    }
    health_check_path = "/health"
  }

  app_settings = {
    ASPNETCORE_ENVIRONMENT = "Staging"
  }
}
```

```bash
# Deploy to staging slot, swap to production when ready
az webapp deployment slot swap \
  --resource-group myapp-prod-rg \
  --name myapp-prod-web \
  --slot staging \
  --target-slot production
```

## Application Insights

```hcl
resource "azurerm_application_insights" "app" {
  name                = "myapp-${var.environment}-insights"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location
  application_type    = "web"
  workspace_id        = azurerm_log_analytics_workspace.main.id
}
```

## Summary

Deploying .NET on Azure App Service with OpenTofu uses managed identity for Key Vault access (no secrets in environment variables), deployment slots for zero-downtime blue-green deployments, Application Insights for observability, and Azure SQL for managed database service. The Key Vault reference syntax in app settings (`@Microsoft.KeyVault(...)`) allows secrets to be rotated in Key Vault without redeploying the application.
