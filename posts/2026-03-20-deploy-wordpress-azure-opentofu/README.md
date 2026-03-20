# How to Deploy a WordPress Site with OpenTofu on Azure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, WordPress, Azure, App Service, MySQL, Infrastructure as Code

Description: Learn how to deploy a production-ready WordPress site on Azure using OpenTofu, with App Service for compute, Azure Database for MySQL, and Azure Files for shared storage.

## Introduction

Deploying WordPress on Azure with OpenTofu uses Azure App Service for managed hosting, Azure Database for MySQL Flexible Server for the database, and Azure Files for shared WordPress content. This combination provides a fully managed, scalable platform with minimal operational overhead.

## Resource Group and Networking

```hcl
resource "azurerm_resource_group" "wordpress" {
  name     = "wordpress-${var.environment}-rg"
  location = var.location
}

resource "azurerm_virtual_network" "wordpress" {
  name                = "wordpress-vnet"
  resource_group_name = azurerm_resource_group.wordpress.name
  location            = azurerm_resource_group.wordpress.location
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "app" {
  name                 = "app-subnet"
  resource_group_name  = azurerm_resource_group.wordpress.name
  virtual_network_name = azurerm_virtual_network.wordpress.name
  address_prefixes     = ["10.0.1.0/24"]

  delegation {
    name = "app-service-delegation"
    service_delegation {
      name    = "Microsoft.Web/serverFarms"
      actions = ["Microsoft.Network/virtualNetworks/subnets/action"]
    }
  }
}
```

## Azure Database for MySQL Flexible Server

```hcl
resource "azurerm_mysql_flexible_server" "wordpress" {
  name                   = "wordpress-${var.environment}-mysql"
  resource_group_name    = azurerm_resource_group.wordpress.name
  location               = azurerm_resource_group.wordpress.location
  administrator_login    = "wpadmin"
  administrator_password = var.db_password
  sku_name               = "GP_Standard_D2ds_v4"
  version                = "8.0.21"
  zone                   = "1"

  storage {
    size_gb = 20
  }

  backup_retention_days        = 7
  geo_redundant_backup_enabled = false

  lifecycle {
    prevent_destroy = true
  }
}

resource "azurerm_mysql_flexible_database" "wordpress" {
  name                = "wordpress"
  resource_group_name = azurerm_resource_group.wordpress.name
  server_name         = azurerm_mysql_flexible_server.wordpress.name
  charset             = "utf8mb4"
  collation           = "utf8mb4_unicode_ci"
}
```

## App Service Plan and Web App

```hcl
resource "azurerm_service_plan" "wordpress" {
  name                = "wordpress-${var.environment}-plan"
  resource_group_name = azurerm_resource_group.wordpress.name
  location            = azurerm_resource_group.wordpress.location
  os_type             = "Linux"
  sku_name            = "P1v3"
}

resource "azurerm_linux_web_app" "wordpress" {
  name                = "wordpress-${var.environment}-app"
  resource_group_name = azurerm_resource_group.wordpress.name
  location            = azurerm_resource_group.wordpress.location
  service_plan_id     = azurerm_service_plan.wordpress.id

  site_config {
    application_stack {
      docker_image_name = "wordpress:6.4-apache"
    }
  }

  app_settings = {
    WORDPRESS_DB_HOST     = "${azurerm_mysql_flexible_server.wordpress.fqdn}:3306"
    WORDPRESS_DB_NAME     = "wordpress"
    WORDPRESS_DB_USER     = "wpadmin"
    WORDPRESS_DB_PASSWORD = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault_secret.db_password.id})"
    WEBSITES_ENABLE_APP_SERVICE_STORAGE = "false"
  }

  identity {
    type = "SystemAssigned"
  }
}
```

## Azure Files for WordPress Content

```hcl
resource "azurerm_storage_account" "wordpress" {
  name                     = "wp${var.environment}storage"
  resource_group_name      = azurerm_resource_group.wordpress.name
  location                 = azurerm_resource_group.wordpress.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_storage_share" "wordpress" {
  name                 = "wp-content"
  storage_account_name = azurerm_storage_account.wordpress.name
  quota                = 100
}
```

## Azure CDN for Performance

```hcl
resource "azurerm_cdn_profile" "wordpress" {
  name                = "wordpress-${var.environment}-cdn"
  resource_group_name = azurerm_resource_group.wordpress.name
  location            = "Global"
  sku                 = "Standard_Microsoft"
}

resource "azurerm_cdn_endpoint" "wordpress" {
  name                = "wordpress-${var.environment}"
  profile_name        = azurerm_cdn_profile.wordpress.name
  resource_group_name = azurerm_resource_group.wordpress.name
  location            = "Global"

  origin {
    name      = "wordpress-origin"
    host_name = azurerm_linux_web_app.wordpress.default_hostname
  }
}
```

## Outputs

```hcl
output "wordpress_url" {
  value       = "https://${azurerm_linux_web_app.wordpress.default_hostname}"
  description = "WordPress site URL"
}

output "cdn_url" {
  value       = "https://${azurerm_cdn_endpoint.wordpress.fqdn}"
  description = "CDN endpoint URL"
}
```

## Summary

Deploying WordPress on Azure with OpenTofu uses Azure App Service with Docker for containerized hosting, Azure Database for MySQL Flexible Server for managed database, Azure Files for shared `wp-content` storage, and Azure CDN for content delivery. Store the database password in Azure Key Vault and reference it in App Service settings using Key Vault references rather than hardcoding it in app settings.
