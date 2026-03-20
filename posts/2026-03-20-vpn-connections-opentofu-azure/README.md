# How to Configure VPN Connections on Azure with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, VPN, VPN Gateway, Networking, Infrastructure as Code

Description: Learn how to provision Azure VPN Gateway connections using OpenTofu to establish Site-to-Site VPN connectivity between Azure VNets and on-premises networks.

## Introduction

Azure VPN Gateway enables secure, encrypted connections between Azure Virtual Networks and on-premises networks. Using OpenTofu to manage VPN Gateway configurations ensures consistent deployment and easy replication across environments.

## Provider Configuration

```hcl
provider "azurerm" {
  features {}
}
```

Resource Group and VNet

```hcl
resource "azurerm_resource_group" "vpn" {
  name     = "vpn-rg"
  location = "East US"
}

resource "azurerm_virtual_network" "main" {
  name                = "main-vnet"
  location            = azurerm_resource_group.vpn.location
  resource_group_name = azurerm_resource_group.vpn.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "gateway" {
  name                 = "GatewaySubnet"   # Must be exactly "GatewaySubnet"
  resource_group_name  = azurerm_resource_group.vpn.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.255.0/27"]
}
```

## Public IP for VPN Gateway

```hcl
resource "azurerm_public_ip" "vpn_gw" {
  name                = "vpn-gateway-pip"
  location            = azurerm_resource_group.vpn.location
  resource_group_name = azurerm_resource_group.vpn.name
  allocation_method   = "Static"
  sku                 = "Standard"
}
```

## VPN Gateway

```hcl
resource "azurerm_virtual_network_gateway" "main" {
  name                = "main-vpn-gateway"
  location            = azurerm_resource_group.vpn.location
  resource_group_name = azurerm_resource_group.vpn.name
  type                = "Vpn"
  vpn_type            = "RouteBased"
  sku                 = "VpnGw1"
  active_active       = false
  enable_bgp          = false

  ip_configuration {
    name                          = "vnetGatewayConfig"
    public_ip_address_id          = azurerm_public_ip.vpn_gw.id
    private_ip_address_allocation = "Dynamic"
    subnet_id                     = azurerm_subnet.gateway.id
  }
}
```

## Local Network Gateway (On-Premises)

```hcl
resource "azurerm_local_network_gateway" "on_prem" {
  name                = "on-prem-lng"
  location            = azurerm_resource_group.vpn.location
  resource_group_name = azurerm_resource_group.vpn.name
  gateway_address     = var.on_prem_public_ip
  address_space       = [var.on_prem_cidr]
}
```

## VPN Connection

```hcl
resource "azurerm_virtual_network_gateway_connection" "main" {
  name                = "main-vpn-connection"
  location            = azurerm_resource_group.vpn.location
  resource_group_name = azurerm_resource_group.vpn.name
  type                       = "IPsec"
  virtual_network_gateway_id = azurerm_virtual_network_gateway.main.id
  local_network_gateway_id   = azurerm_local_network_gateway.on_prem.id
  shared_key                 = var.vpn_shared_key
}
```

## Deploying

```bash
tofu init
tofu plan
tofu apply
```

Note: Azure VPN Gateway deployment takes 30-45 minutes.

## Conclusion

OpenTofu manages the full Azure VPN Gateway stack - from the gateway subnet and public IP to the Local Network Gateway and connection resource. Managing this as code ensures consistent VPN deployments and makes it easy to replicate across regions or environments.
