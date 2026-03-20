# How to Configure IPv6 in Azure Virtual Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, IPv6, Virtual Network, VNet, Dual-Stack, Cloud Networking

Description: Configure IPv6 address spaces and subnets in Azure Virtual Networks, creating dual-stack VNets that support both IPv4 and IPv6 workloads.

## Introduction

Azure Virtual Networks support IPv6 through dual-stack configuration: you add an IPv6 address space alongside the existing IPv4 address space. Azure uses `/48` IPv6 prefixes for VNets and `/64` for subnets. Azure IPv6 addresses are globally unique and routable, similar to AWS. This guide covers creating and configuring dual-stack Azure VNets.

## Create Dual-Stack VNet with Azure CLI

```bash
RESOURCE_GROUP="rg-ipv6-network"
LOCATION="eastus"

# Create resource group

az group create --name "$RESOURCE_GROUP" --location "$LOCATION"

# Create VNet with both IPv4 and IPv6 address spaces
az network vnet create \
    --resource-group "$RESOURCE_GROUP" \
    --name vnet-dualstack \
    --address-prefixes "10.0.0.0/16" "fd00:db8::/48" \
    --location "$LOCATION"

# Add IPv6 subnet to existing VNet
az network vnet subnet create \
    --resource-group "$RESOURCE_GROUP" \
    --vnet-name vnet-dualstack \
    --name subnet-web \
    --address-prefixes "10.0.1.0/24" "fd00:db8:0:1::/64"

# Add private subnet with IPv6
az network vnet subnet create \
    --resource-group "$RESOURCE_GROUP" \
    --vnet-name vnet-dualstack \
    --name subnet-app \
    --address-prefixes "10.0.2.0/24" "fd00:db8:0:2::/64"

# Verify VNet IPv6 configuration
az network vnet show \
    --resource-group "$RESOURCE_GROUP" \
    --name vnet-dualstack \
    --query "{name:name, addressSpace:addressSpace, subnets:subnets[*].{name:name, prefixes:addressPrefixes}}"
```

## Terraform Azure VNet with IPv6

```hcl
# vnet_ipv6.tf

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

resource "azurerm_resource_group" "main" {
  name     = "rg-ipv6-network"
  location = "East US"
}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-dualstack"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  # Dual-stack: IPv4 + IPv6 address spaces
  address_space = [
    "10.0.0.0/16",      # IPv4
    "fd00:db8::/48",    # IPv6 (ULA)
  ]

  tags = { environment = "production" }
}

resource "azurerm_subnet" "web" {
  name                 = "subnet-web"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name

  address_prefixes = [
    "10.0.1.0/24",          # IPv4
    "fd00:db8:0:1::/64",    # IPv6
  ]
}

resource "azurerm_subnet" "app" {
  name                 = "subnet-app"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name

  address_prefixes = [
    "10.0.2.0/24",
    "fd00:db8:0:2::/64",
  ]
}

resource "azurerm_subnet" "db" {
  name                 = "subnet-db"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name

  address_prefixes = [
    "10.0.3.0/24",
    "fd00:db8:0:3::/64",
  ]
}
```

## Azure VNet IPv6 Considerations

```bash
# Azure IPv6 differences from IPv4:
# 1. VNet peering supports IPv6 (dual-stack to dual-stack)
# 2. IPv6 ULA (fd00::/8) addresses are supported
# 3. Can also use globally routable IPv6 via public IP prefix
# 4. ExpressRoute supports IPv6
# 5. VPN Gateway supports IPv6 dual-stack

# Add IPv6 address space to existing VNet
az network vnet update \
    --resource-group "$RESOURCE_GROUP" \
    --name existing-vnet \
    --add addressSpace.addressPrefixes "fd00:existing::/48"

# Update subnet to add IPv6
az network vnet subnet update \
    --resource-group "$RESOURCE_GROUP" \
    --vnet-name existing-vnet \
    --name existing-subnet \
    --add addressPrefixes "fd00:existing:0:1::/64"
```

## VNet Peering with IPv6

```hcl
# Peer two dual-stack VNets
resource "azurerm_virtual_network_peering" "a_to_b" {
  name                      = "peer-a-to-b"
  resource_group_name       = azurerm_resource_group.main.name
  virtual_network_name      = azurerm_virtual_network.vnet_a.name
  remote_virtual_network_id = azurerm_virtual_network.vnet_b.id

  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = false
  use_remote_gateways          = false
}
```

## Conclusion

Azure dual-stack VNets combine IPv4 and IPv6 address spaces in the same virtual network. Add IPv6 address spaces (either globally routable or ULA `fd00::/8`) alongside IPv4 prefixes. Subnets get both IPv4 and IPv6 CIDR blocks. Most Azure networking features (NSGs, route tables, load balancers) support IPv6 when properly configured. Start with ULA addresses for internal-only IPv6, or use public IP prefixes for globally routable IPv6 addresses visible from the internet.
