# How to Deploy Azure SQL Database with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, SQL Database, Infrastructure as Code, Database, DevOps

Description: Learn how to provision an Azure SQL Database server and database with firewall rules and transparent data encryption using OpenTofu.

---

Azure SQL Database is a fully managed relational database as a service. OpenTofu's `azurerm` provider lets you declare SQL servers, databases, and associated security settings as version-controlled infrastructure.

---

## Create a SQL Server

```hcl
resource "azurerm_resource_group" "db" {
  name     = "db-rg"
  location = "eastus"
}

resource "azurerm_mssql_server" "main" {
  name                         = "my-sql-server-prod"
  resource_group_name          = azurerm_resource_group.db.name
  location                     = azurerm_resource_group.db.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.db_admin_password

  minimum_tls_version = "1.2"

  azuread_administrator {
    login_username = "aad-admin"
    object_id      = var.aad_admin_object_id
  }

  tags = {
    Environment = "production"
  }
}
```

---

## Create the Database

```hcl
resource "azurerm_mssql_database" "app" {
  name        = "appdb"
  server_id   = azurerm_mssql_server.main.id
  collation   = "SQL_Latin1_General_CP1_CI_AS"
  max_size_gb = 32
  sku_name    = "GP_S_Gen5_2"  # General Purpose Serverless

  auto_pause_delay_in_minutes = 60
  min_capacity                = 0.5

  tags = {
    Application = "my-app"
  }
}
```

---

## Configure Firewall Rules

```hcl
# Allow Azure services

resource "azurerm_mssql_firewall_rule" "azure_services" {
  name             = "allow-azure-services"
  server_id        = azurerm_mssql_server.main.id
  start_ip_address = "0.0.0.0"
  end_ip_address   = "0.0.0.0"
}

# Allow specific office IP
resource "azurerm_mssql_firewall_rule" "office" {
  name             = "office-ip"
  server_id        = azurerm_mssql_server.main.id
  start_ip_address = var.office_ip
  end_ip_address   = var.office_ip
}
```

---

## Private Endpoint for Secure Access

```hcl
resource "azurerm_private_endpoint" "sql" {
  name                = "sql-private-endpoint"
  resource_group_name = azurerm_resource_group.db.name
  location            = azurerm_resource_group.db.location
  subnet_id           = azurerm_subnet.private.id

  private_service_connection {
    name                           = "sql-connection"
    private_connection_resource_id = azurerm_mssql_server.main.id
    subresource_names              = ["sqlServer"]
    is_manual_connection           = false
  }
}
```

---

## Summary

Use `azurerm_mssql_server` for the logical server (authentication, AAD admin, TLS settings) and `azurerm_mssql_database` for the actual database (SKU, size, collation). Add firewall rules with `azurerm_mssql_firewall_rule` and use Private Endpoints to eliminate internet exposure for production databases.
