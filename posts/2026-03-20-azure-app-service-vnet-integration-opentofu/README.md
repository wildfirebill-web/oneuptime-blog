# How to Configure Azure App Service VNet Integration with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, App Service, VNet Integration, OpenTofu, Networking, Security

Description: Learn how to configure Azure App Service VNet Integration with OpenTofu to enable web apps to securely communicate with resources in an Azure virtual network.

## Overview

Azure App Service VNet Integration enables outbound connectivity from your web app to resources inside a virtual network. This is essential for accessing databases, storage, or other services on private networks without exposing them to the internet.

## Step 1: Create VNet and Subnet for Integration

```hcl
# main.tf - VNet and delegation subnet for App Service VNet integration
resource "azurerm_virtual_network" "vnet" {
  name                = "app-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Subnet for App Service VNet integration - requires /28 or larger
resource "azurerm_subnet" "app_service_subnet" {
  name                 = "app-service-integration-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.5.0/28"]

  # Required delegation for App Service VNet integration
  delegation {
    name = "app-service-delegation"
    service_delegation {
      name = "Microsoft.Web/serverFarms"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/action",
      ]
    }
  }
}

# Subnet for backend resources (e.g., databases)
resource "azurerm_subnet" "backend_subnet" {
  name                 = "backend-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}
```

## Step 2: Create App Service Plan and Web App

```hcl
# Premium or Standard plan is required for VNet integration
resource "azurerm_service_plan" "plan" {
  name                = "vnet-integrated-plan"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  os_type             = "Linux"
  sku_name            = "P1v3"  # Premium V3 supports VNet integration
}

resource "azurerm_linux_web_app" "app" {
  name                = "my-vnet-integrated-app"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  service_plan_id     = azurerm_service_plan.plan.id
  https_only          = true

  site_config {
    application_stack {
      node_version = "20-lts"
    }

    # Route all outbound traffic through VNet
    vnet_route_all_enabled = true
  }

  # Attach to VNet integration subnet
  virtual_network_subnet_id = azurerm_subnet.app_service_subnet.id
}
```

## Step 3: Enable Access to Private Resources

```hcl
# PostgreSQL Flexible Server on the backend subnet
resource "azurerm_postgresql_flexible_server" "db" {
  name                   = "my-private-postgres"
  resource_group_name    = azurerm_resource_group.rg.name
  location               = azurerm_resource_group.rg.location
  administrator_login    = "pgadmin"
  administrator_password = var.db_password
  version                = "15"
  sku_name               = "B_Standard_B1ms"

  # Deploy to the backend subnet
  delegated_subnet_id = azurerm_subnet.backend_subnet.id
  private_dns_zone_id = azurerm_private_dns_zone.postgres_dns.id
}
```

## Step 4: Private Endpoint for Inbound Access

```hcl
# Private endpoint to make the App Service accessible from within the VNet
resource "azurerm_private_endpoint" "app_pe" {
  name                = "app-private-endpoint"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  subnet_id           = azurerm_subnet.backend_subnet.id

  private_service_connection {
    name                           = "app-private-connection"
    private_connection_resource_id = azurerm_linux_web_app.app.id
    is_manual_connection           = false
    subresource_names              = ["sites"]
  }
}
```

## Step 5: Outputs

```hcl
output "app_vnet_integration_subnet" {
  value       = azurerm_subnet.app_service_subnet.id
  description = "Subnet used for App Service VNet integration"
}

output "app_url" {
  value = "https://${azurerm_linux_web_app.app.default_hostname}"
}
```

## Summary

Azure App Service VNet Integration with OpenTofu enables secure private connectivity between your web app and backend resources like databases and storage. With `vnet_route_all_enabled`, all outbound traffic routes through the VNet, ensuring backend resources are never accessed over the public internet.
