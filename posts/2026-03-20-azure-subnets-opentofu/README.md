# How to Create Subnets with OpenTofu on Azure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Subnets, Networking, Infrastructure as Code

Description: Learn how to create Azure subnets with OpenTofu including service endpoints, delegation, and network security group associations.

## Introduction

Subnets divide an Azure VNet into smaller address ranges. Resources like VMs, AKS nodes, and App Service plans are deployed into subnets. OpenTofu manages subnets with service endpoints and delegations for PaaS service integration.

## Basic Subnets

```hcl
resource "azurerm_subnet" "public" {
  name                 = "snet-public-${var.environment}"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "private" {
  name                 = "snet-private-${var.environment}"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/24"]

  # Service endpoints enable private access to Azure PaaS services
  service_endpoints = ["Microsoft.Sql", "Microsoft.Storage", "Microsoft.KeyVault"]
}
```

## AKS Subnet with Node Pool Delegation

```hcl
resource "azurerm_subnet" "aks_nodes" {
  name                 = "snet-aks-nodes"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.10.0/22"]  # /22 allows up to 1022 node IPs

  service_endpoints = ["Microsoft.ContainerRegistry"]
}
```

## App Service Delegation Subnet

```hcl
resource "azurerm_subnet" "app_service" {
  name                 = "snet-appservice"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.20.0/24"]

  # Delegate this subnet to App Service for VNet Integration
  delegation {
    name = "app-service-delegation"
    service_delegation {
      name    = "Microsoft.Web/serverFarms"
      actions = ["Microsoft.Network/virtualNetworks/subnets/action"]
    }
  }
}
```

## NSG Association

```hcl
resource "azurerm_network_security_group" "private" {
  name                = "nsg-private-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
}

resource "azurerm_subnet_network_security_group_association" "private" {
  subnet_id                 = azurerm_subnet.private.id
  network_security_group_id = azurerm_network_security_group.private.id
}
```

## Multiple Subnets with For_Each

```hcl
variable "subnets" {
  type = map(object({
    address_prefix     = string
    service_endpoints  = list(string)
  }))
  default = {
    web  = { address_prefix = "10.0.1.0/24", service_endpoints = [] }
    app  = { address_prefix = "10.0.2.0/24", service_endpoints = ["Microsoft.Sql"] }
    data = { address_prefix = "10.0.3.0/24", service_endpoints = ["Microsoft.Sql", "Microsoft.Storage"] }
  }
}

resource "azurerm_subnet" "all" {
  for_each             = var.subnets
  name                 = "snet-${each.key}"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [each.value.address_prefix]
  service_endpoints    = each.value.service_endpoints
}
```

## Outputs

```hcl
output "public_subnet_id"  { value = azurerm_subnet.public.id }
output "private_subnet_id" { value = azurerm_subnet.private.id }
output "all_subnet_ids"    { value = { for k, v in azurerm_subnet.all : k => v.id } }
```

## Conclusion

Plan your subnet address spaces carefully before deployment-Azure does not allow changing subnet address prefixes after resources are deployed. Use service endpoints to enable private access to Azure PaaS services and service delegation for integration with Azure-native services like App Service.
