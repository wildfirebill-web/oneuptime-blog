# How to Create ACR Geo-Replication with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Container Registry, Geo-Replication, Infrastructure as Code

Description: Learn how to configure Azure Container Registry geo-replication with OpenTofu to replicate container images across Azure regions for global deployments.

ACR geo-replication replicates your container registry to multiple Azure regions, enabling fast image pulls from nearby regions and providing redundancy. Managing replication in OpenTofu ensures consistent replica configuration across all target regions.

## Provider Configuration

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}
```

## ACR with Geo-Replication

```hcl
resource "azurerm_resource_group" "registry" {
  name     = "registry-rg"
  location = "eastus"  # Primary region
}

# Premium SKU required for geo-replication

resource "azurerm_container_registry" "global" {
  name                = "myappregistryglobal"
  resource_group_name = azurerm_resource_group.registry.name
  location            = azurerm_resource_group.registry.location
  sku                 = "Premium"
  admin_enabled       = false

  tags = {
    Environment = "production"
  }
}

# Replicate to West Europe
resource "azurerm_container_registry_replication" "europe" {
  name                    = "westeurope"
  resource_group_name     = azurerm_resource_group.registry.name
  container_registry_name = azurerm_container_registry.global.name
  location                = "westeurope"

  # Zone redundancy in this replica
  zone_redundancy_enabled = true

  tags = {
    Region = "westeurope"
  }
}

# Replicate to Southeast Asia
resource "azurerm_container_registry_replication" "asia" {
  name                    = "southeastasia"
  resource_group_name     = azurerm_resource_group.registry.name
  container_registry_name = azurerm_container_registry.global.name
  location                = "southeastasia"

  zone_redundancy_enabled = false  # Zone redundancy not available in all regions
}

# Replicate to Australia East
resource "azurerm_container_registry_replication" "australia" {
  name                    = "australiaeast"
  resource_group_name     = azurerm_resource_group.registry.name
  container_registry_name = azurerm_container_registry.global.name
  location                = "australiaeast"
}
```

## Multiple Replicas with for_each

```hcl
locals {
  replica_regions = {
    "westeurope"    = { zone_redundancy = true  }
    "southeastasia" = { zone_redundancy = false }
    "australiaeast" = { zone_redundancy = false }
    "brazilsouth"   = { zone_redundancy = false }
    "japaneast"     = { zone_redundancy = true  }
  }
}

resource "azurerm_container_registry_replication" "replicas" {
  for_each = local.replica_regions

  name                    = each.key
  resource_group_name     = azurerm_resource_group.registry.name
  container_registry_name = azurerm_container_registry.global.name
  location                = each.key

  zone_redundancy_enabled = each.value.zone_redundancy

  tags = {
    Region = each.key
  }
}
```

## Webhook for Replication Events

```hcl
resource "azurerm_container_registry_webhook" "replication_complete" {
  name                = "replication-webhook"
  resource_group_name = azurerm_resource_group.registry.name
  registry_name       = azurerm_container_registry.global.name
  location            = azurerm_resource_group.registry.location

  service_uri = var.webhook_endpoint
  status      = "enabled"
  scope       = "myapp:*"

  actions = ["push"]

  custom_headers = {
    "X-Registry" = azurerm_container_registry.global.name
  }
}
```

## AKS Integration Per Region

```hcl
# Each regional AKS cluster gets pull access
resource "azurerm_role_assignment" "aks_pull" {
  for_each = {
    eu  = azurerm_kubernetes_cluster.europe.kubelet_identity[0].object_id
    ap  = azurerm_kubernetes_cluster.asia.kubelet_identity[0].object_id
  }

  scope                = azurerm_container_registry.global.id
  role_definition_name = "AcrPull"
  principal_id         = each.value
}
```

## Conclusion

ACR geo-replication in OpenTofu ensures fast container image pulls from any region and provides registry resilience. Use for_each to manage multiple replica regions uniformly, enable zone redundancy in regions that support it for within-region HA, and grant AcrPull to each regional AKS cluster's kubelet identity. The single login_server URL works globally - Azure routes pull requests to the nearest replica automatically.
