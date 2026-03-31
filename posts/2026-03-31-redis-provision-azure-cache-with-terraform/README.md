# How to Provision Azure Cache for Redis with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Terraform, Azure, Azure Cache For Redis, Infrastructure As Code, Cloud

Description: Provision Azure Cache for Redis using Terraform with the azurerm provider, configuring SKU, geo-replication, firewall rules, and private endpoints for secure access.

---

## Prerequisites

- Terraform >= 1.5
- Azure CLI authenticated (`az login`)
- `azurerm` provider configured

## Provider Configuration

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.90"
    }
  }
}

provider "azurerm" {
  features {}
}
```

## Resource Group and Azure Cache for Redis

```hcl
resource "azurerm_resource_group" "rg" {
  name     = "rg-redis-prod"
  location = "East US"
}

resource "azurerm_redis_cache" "redis" {
  name                = "myapp-redis-prod"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  sku_name            = "Premium"
  family              = "P"
  capacity            = 1

  enable_non_ssl_port = false
  minimum_tls_version = "1.2"
  public_network_access_enabled = false

  redis_configuration {
    maxmemory_reserved = 125
    maxmemory_delta    = 125
    maxmemory_policy   = "allkeys-lru"
    rdb_backup_enabled            = true
    rdb_backup_frequency          = 60
    rdb_backup_max_snapshot_count = 2
    rdb_storage_connection_string = azurerm_storage_account.backup.primary_blob_connection_string
    notify_keyspace_events        = ""
  }

  tags = {
    Environment = "production"
    App         = "myapp"
  }
}
```

## Storage Account for RDB Backups

```hcl
resource "azurerm_storage_account" "backup" {
  name                     = "myappredisbackup"
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```

## Firewall Rules for Allowed IP Ranges

```hcl
resource "azurerm_redis_firewall_rule" "office" {
  name                = "office-ip"
  redis_cache_name    = azurerm_redis_cache.redis.name
  resource_group_name = azurerm_resource_group.rg.name
  start_ip            = "203.0.113.10"
  end_ip              = "203.0.113.10"
}

resource "azurerm_redis_firewall_rule" "vpn" {
  name                = "vpn-range"
  redis_cache_name    = azurerm_redis_cache.redis.name
  resource_group_name = azurerm_resource_group.rg.name
  start_ip            = "10.0.0.0"
  end_ip              = "10.0.0.255"
}
```

## Private Endpoint (Premium SKU Only)

```hcl
resource "azurerm_virtual_network" "vnet" {
  name                = "vnet-prod"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "private" {
  name                 = "subnet-private"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
  private_endpoint_network_policies = "Disabled"
}

resource "azurerm_private_endpoint" "redis" {
  name                = "pe-redis"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  subnet_id           = azurerm_subnet.private.id

  private_service_connection {
    name                           = "redis-privateserviceconnection"
    private_connection_resource_id = azurerm_redis_cache.redis.id
    subresource_names              = ["redisCache"]
    is_manual_connection           = false
  }
}
```

## Outputs

```hcl
output "redis_hostname" {
  value = azurerm_redis_cache.redis.hostname
}

output "redis_primary_key" {
  value     = azurerm_redis_cache.redis.primary_access_key
  sensitive = true
}

output "redis_port" {
  value = azurerm_redis_cache.redis.ssl_port
}

output "redis_connection_string" {
  value     = "${azurerm_redis_cache.redis.hostname}:${azurerm_redis_cache.redis.ssl_port},password=${azurerm_redis_cache.redis.primary_access_key},ssl=True,abortConnect=False"
  sensitive = true
}
```

## Deploy

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

Retrieve connection info:

```bash
terraform output -raw redis_hostname
terraform output -raw redis_primary_key
```

## Geo-Replication (Premium SKU)

```hcl
resource "azurerm_redis_cache" "secondary" {
  name                = "myapp-redis-secondary"
  location            = "West Europe"
  resource_group_name = azurerm_resource_group.rg.name
  sku_name            = "Premium"
  family              = "P"
  capacity            = 1
  enable_non_ssl_port = false
  minimum_tls_version = "1.2"
}

resource "azurerm_redis_linked_server" "geo" {
  target_redis_cache_name     = azurerm_redis_cache.redis.name
  resource_group_name         = azurerm_resource_group.rg.name
  linked_redis_cache_id       = azurerm_redis_cache.secondary.id
  linked_redis_cache_location = azurerm_redis_cache.secondary.location
  server_role                 = "Secondary"
}
```

## Summary

Terraform provisions Azure Cache for Redis with the `azurerm_redis_cache` resource, supporting standalone and geo-replicated topologies. Premium SKU enables private endpoints, RDB backups to Azure Storage, and geo-replication. Sensitive outputs like the access key are marked sensitive to prevent exposure in logs.
