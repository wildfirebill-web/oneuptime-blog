# How to Configure Azure NSG Rules for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, IPv6, NSG, Network Security Group, Firewall, Dual-Stack

Description: Create Azure Network Security Group rules for IPv6 traffic, understand how NSG rules apply to dual-stack subnets, and configure security policies for IPv6 VMs.

## Introduction

Azure Network Security Groups (NSGs) filter traffic by IP address ranges, ports, and protocols. For IPv6, you must add specific rules allowing or denying IPv6 address ranges - there is no automatic inheritance from IPv4 rules. NSGs apply to subnets or individual NICs and are the primary Layer 4 firewall mechanism in Azure virtual networks.

## Create NSG with IPv6 Rules

```bash
RG="rg-ipv6"

# Create NSG

az network nsg create \
    --resource-group "$RG" \
    --name nsg-web-ipv6

# Allow HTTP from all IPv6
az network nsg rule create \
    --resource-group "$RG" \
    --nsg-name nsg-web-ipv6 \
    --name AllowHTTPv6Inbound \
    --priority 200 \
    --protocol Tcp \
    --direction Inbound \
    --source-address-prefixes "::/0" \
    --source-port-ranges "*" \
    --destination-address-prefixes "*" \
    --destination-port-ranges 80 \
    --access Allow

# Allow HTTPS from all IPv6
az network nsg rule create \
    --resource-group "$RG" \
    --nsg-name nsg-web-ipv6 \
    --name AllowHTTPSv6Inbound \
    --priority 210 \
    --protocol Tcp \
    --direction Inbound \
    --source-address-prefixes "::/0" \
    --source-port-ranges "*" \
    --destination-address-prefixes "*" \
    --destination-port-ranges 443 \
    --access Allow

# Allow SSH from specific IPv6 prefix
az network nsg rule create \
    --resource-group "$RG" \
    --nsg-name nsg-web-ipv6 \
    --name AllowSSHv6Inbound \
    --priority 300 \
    --protocol Tcp \
    --direction Inbound \
    --source-address-prefixes "2001:db8:admin::/48" \
    --source-port-ranges "*" \
    --destination-address-prefixes "*" \
    --destination-port-ranges 22 \
    --access Allow

# Associate NSG with subnet
az network vnet subnet update \
    --resource-group "$RG" \
    --vnet-name vnet-dualstack \
    --name subnet-web \
    --network-security-group nsg-web-ipv6
```

## Terraform NSG with IPv6 Rules

```hcl
# nsg_ipv6.tf

resource "azurerm_network_security_group" "web" {
  name                = "nsg-web"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  # Allow HTTP - IPv4
  security_rule {
    name                       = "AllowHTTPv4"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  # Allow HTTP - IPv6
  security_rule {
    name                       = "AllowHTTPv6"
    priority                   = 101
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "::/0"  # IPv6 any
    destination_address_prefix = "*"
  }

  # Allow HTTPS - IPv4
  security_rule {
    name                       = "AllowHTTPSv4"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  # Allow HTTPS - IPv6
  security_rule {
    name                       = "AllowHTTPSv6"
    priority                   = 111
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "::/0"
    destination_address_prefix = "*"
  }

  # Allow ICMPv6 (for NDP and ping6)
  security_rule {
    name                       = "AllowICMPv6"
    priority                   = 200
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Icmp"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "::/0"
    destination_address_prefix = "*"
  }

  # Deny all other IPv6 inbound
  security_rule {
    name                       = "DenyAllIPv6Inbound"
    priority                   = 4000
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "::/0"
    destination_address_prefix = "*"
  }

  tags = { Name = "web-nsg" }
}

# Associate NSG with subnet
resource "azurerm_subnet_network_security_group_association" "web" {
  subnet_id                 = azurerm_subnet.web.id
  network_security_group_id = azurerm_network_security_group.web.id
}
```

## NSG Default Rules for IPv6

```bash
# Azure NSGs have default rules that apply to all traffic
# Default rules (cannot be deleted, only overridden with lower priority number):
# 65000 - AllowVNetInBound: Allow from VNet (covers IPv6 VNet ranges too)
# 65001 - AllowAzureLoadBalancerInBound: Allow ALB health probes
# 65500 - DenyAllInBound: Deny all other inbound

# For IPv6 from internet (not VNet), you need explicit allow rules
# because the default DenyAllInBound blocks external IPv6 traffic
```

## Conclusion

Azure NSG rules for IPv6 require explicit entries with IPv6 source or destination prefixes (`::/0` for any IPv6). There is no automatic coverage from IPv4 rules - `*` in a rule means "any address" but IPv6 addresses from the internet need explicit `::/0` rules. Always add ICMPv6 allow rules to enable NDP and connectivity testing. Associate NSGs to subnets (not individual NICs when possible) for centralized security management. Use consecutive priority numbers for paired IPv4/IPv6 rules (100 for IPv4, 101 for IPv6) to keep rules organized.
