# How to Configure Azure VPN Gateway with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, IPv6, VPN Gateway, Site-to-Site, Point-to-Site, Dual-Stack

Description: Configure Azure VPN Gateway with IPv6 for site-to-site VPN connections, enabling dual-stack connectivity between Azure VNets and on-premises networks over IPv6.

## Introduction

Azure VPN Gateway supports IPv6 for both site-to-site (S2S) and point-to-site (P2S) VPN connections. For S2S, you can establish VPN tunnels over IPv6 transport. For P2S, you can assign IPv6 addresses to VPN clients from a custom IPv6 address pool. This enables IPv6 connectivity extension from on-premises networks into Azure virtual networks.

## Create Dual-Stack VPN Gateway

```bash
RG="rg-vpn"
LOCATION="eastus"

# VNet and GatewaySubnet with IPv6

az network vnet create \
    --resource-group "$RG" \
    --name vnet-vpn \
    --address-prefixes "10.1.0.0/16" "fd00:vpn::/48"

# GatewaySubnet must be at least /27 for IPv4
az network vnet subnet create \
    --resource-group "$RG" \
    --vnet-name vnet-vpn \
    --name GatewaySubnet \
    --address-prefixes "10.1.0.0/27" "fd00:vpn:0:1::/64"

# Create IPv4 public IP for VPN gateway
az network public-ip create \
    --resource-group "$RG" \
    --name pip-vpngw-ipv4 \
    --version IPv4 \
    --sku Standard \
    --allocation-method Static

# Create IPv6 public IP for VPN gateway
az network public-ip create \
    --resource-group "$RG" \
    --name pip-vpngw-ipv6 \
    --version IPv6 \
    --sku Standard \
    --allocation-method Static

# Create dual-stack VPN Gateway
az network vnet-gateway create \
    --resource-group "$RG" \
    --name vpngw-dualstack \
    --vnet vnet-vpn \
    --public-ip-addresses pip-vpngw-ipv4 pip-vpngw-ipv6 \
    --gateway-type Vpn \
    --vpn-type RouteBased \
    --sku VpnGw2 \
    --no-wait

echo "VPN Gateway creation started (takes 20-45 minutes)..."
az network vnet-gateway wait \
    --resource-group "$RG" \
    --name vpngw-dualstack \
    --created
```

## Terraform VPN Gateway with IPv6

```hcl
# vpn_gateway_ipv6.tf

resource "azurerm_public_ip" "vpngw_ipv4" {
  name                = "pip-vpngw-ipv4"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
  ip_version          = "IPv4"
}

resource "azurerm_public_ip" "vpngw_ipv6" {
  name                = "pip-vpngw-ipv6"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
  ip_version          = "IPv6"
}

resource "azurerm_virtual_network_gateway" "main" {
  name                = "vpngw-dualstack"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  type     = "Vpn"
  vpn_type = "RouteBased"
  sku      = "VpnGw2"
  generation = "Generation2"

  active_active = false
  enable_bgp    = false

  # IPv4 and IPv6 public IPs
  ip_configuration {
    name                          = "ipconfig-ipv4"
    public_ip_address_id          = azurerm_public_ip.vpngw_ipv4.id
    private_ip_address_allocation = "Dynamic"
    subnet_id                     = azurerm_subnet.gateway.id
  }

  # Point-to-site with IPv6 client address pool
  vpn_client_configuration {
    address_space = ["172.16.0.0/24", "fd00:clients::/64"]  # IPv4 + IPv6 pool

    vpn_client_protocols = ["OpenVPN"]

    aad_tenant   = "https://login.microsoftonline.com/${var.tenant_id}/"
    aad_audience = "41b23e61-6c1e-4545-b367-cd054e0ed4b4"
    aad_issuer   = "https://sts.windows.net/${var.tenant_id}/"
  }
}

# Site-to-site connection with IPv6
resource "azurerm_local_network_gateway" "onprem" {
  name                = "lgw-onprem"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  gateway_address = "203.0.113.1"  # On-premises public IP

  address_space = [
    "192.168.0.0/24",    # On-premises IPv4
    "fd00:onprem::/48",  # On-premises IPv6
  ]
}

resource "azurerm_virtual_network_gateway_connection" "s2s" {
  name                = "conn-s2s"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  type                            = "IPsec"
  virtual_network_gateway_id      = azurerm_virtual_network_gateway.main.id
  local_network_gateway_id        = azurerm_local_network_gateway.onprem.id
  connection_protocol             = "IKEv2"
  shared_key                      = var.vpn_shared_key

  # Enable IPv6 traffic selector
  traffic_selector_policy {
    local_address_cidr  = ["10.1.0.0/16", "fd00:vpn::/48"]
    remote_address_cidr = ["192.168.0.0/24", "fd00:onprem::/48"]
  }
}
```

## Verify VPN Gateway IPv6

```bash
# Get VPN gateway public IPs
az network vnet-gateway show \
    --resource-group "$RG" \
    --name vpngw-dualstack \
    --query "ipConfigurations[*].{name:name, publicIP:publicIPAddress.id}"

# Check connection status
az network vpn-connection show \
    --resource-group "$RG" \
    --name conn-s2s \
    --query "{status:connectionStatus, protocol:connectionProtocol}"
```

## Conclusion

Azure VPN Gateway dual-stack supports IPv6 for both control plane (P2P VPN tunnel establishment) and data plane (IPv6 traffic through the tunnel). Deploy with two public IPs (one IPv4, one IPv6) for the gateway. For S2S connections, add IPv6 address spaces to the local network gateway representing on-premises networks. For P2S, add an IPv6 address pool in the VPN client configuration. Azure VPN Gateway requires the VpnGw2 SKU or higher for dual-stack IPv6 support.
