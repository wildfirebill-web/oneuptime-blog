# How to Configure Azure Firewall with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, IPv6, Azure Firewall, Network Security, Dual-Stack, Hub-Spoke

Description: Configure Azure Firewall with IPv6 support for hub-spoke network architectures, creating network rules that allow and deny IPv6 traffic through the centralized firewall.

## Introduction

Azure Firewall Premium and Standard support IPv6 network rules, allowing you to filter IPv6 traffic in hub-spoke VNet architectures. Azure Firewall can inspect and filter both IPv4 and IPv6 traffic, and you can create explicit allow/deny rules for IPv6 source and destination addresses. This is important for organizations deploying dual-stack Azure networks with centralized security inspection.

## Deploy Azure Firewall with IPv6

```bash
RG="rg-firewall"
LOCATION="eastus"

# Hub VNet with dual-stack
az network vnet create \
    --resource-group "$RG" \
    --name vnet-hub \
    --address-prefixes "10.0.0.0/16" "fd00:hub::/48"

# AzureFirewallSubnet (required name, /26 minimum)
az network vnet subnet create \
    --resource-group "$RG" \
    --vnet-name vnet-hub \
    --name AzureFirewallSubnet \
    --address-prefixes "10.0.0.0/26" "fd00:hub:0:1::/64"

# Create IPv4 public IP for firewall
az network public-ip create \
    --resource-group "$RG" \
    --name pip-fw-ipv4 \
    --version IPv4 \
    --sku Standard

# Create Azure Firewall with dual-stack
az network firewall create \
    --resource-group "$RG" \
    --name fw-hub \
    --sku AZFW_VNet \
    --tier Premium \
    --vnet-name vnet-hub \
    --public-ip pip-fw-ipv4

# Add IPv6 network rule
az network firewall network-rule create \
    --resource-group "$RG" \
    --firewall-name fw-hub \
    --collection-name "allow-ipv6-outbound" \
    --name "allow-http-ipv6" \
    --priority 100 \
    --action Allow \
    --protocols TCP \
    --source-addresses "fd00:spoke::/48" \
    --destination-addresses "::/0" \
    --destination-ports 80 443
```

## Terraform Azure Firewall with IPv6

```hcl
# azure_firewall_ipv6.tf

resource "azurerm_public_ip" "firewall_ipv4" {
  name                = "pip-fw-ipv4"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
  ip_version          = "IPv4"
}

resource "azurerm_firewall" "hub" {
  name                = "fw-hub"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku_name            = "AZFW_VNet"
  sku_tier            = "Premium"

  ip_configuration {
    name                 = "fw-ipconfig"
    subnet_id            = azurerm_subnet.firewall.id
    public_ip_address_id = azurerm_public_ip.firewall_ipv4.id
  }

  tags = { Name = "hub-firewall" }
}

# Firewall Policy with IPv6 rules
resource "azurerm_firewall_policy" "main" {
  name                = "fw-policy"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Premium"
}

resource "azurerm_firewall_policy_rule_collection_group" "main" {
  name               = "main-rules"
  firewall_policy_id = azurerm_firewall_policy.main.id
  priority           = 100

  network_rule_collection {
    name     = "allow-ipv6-outbound"
    priority = 100
    action   = "Allow"

    rule {
      name              = "allow-http-ipv6"
      protocols         = ["TCP"]
      source_addresses  = ["fd00:spoke::/48"]
      destination_addresses = ["::/0"]
      destination_ports = ["80", "443"]
    }

    rule {
      name              = "allow-dns-ipv6"
      protocols         = ["TCP", "UDP"]
      source_addresses  = ["fd00:spoke::/48"]
      destination_addresses = ["::/0"]
      destination_ports = ["53"]
    }

    rule {
      name              = "allow-icmpv6"
      protocols         = ["ICMP"]
      source_addresses  = ["::/0"]
      destination_addresses = ["fd00:hub::/48"]
      destination_ports = ["*"]
    }
  }

  network_rule_collection {
    name     = "deny-ipv6-inbound"
    priority = 200
    action   = "Deny"

    rule {
      name                  = "deny-all-ipv6-inbound"
      protocols             = ["Any"]
      source_addresses      = ["::/0"]
      destination_addresses = ["fd00:spoke::/48"]
      destination_ports     = ["*"]
    }
  }
}
```

## Route Traffic Through Firewall

```bash
# Route table to send spoke IPv6 traffic through firewall
FW_IPV4=$(az network firewall show \
    --resource-group "$RG" \
    --name fw-hub \
    --query "ipConfigurations[0].privateIPAddress" \
    --output text)

az network route-table create \
    --resource-group "$RG" \
    --name rt-spoke-ipv6

az network route-table route create \
    --resource-group "$RG" \
    --route-table-name rt-spoke-ipv6 \
    --name route-ipv4-to-fw \
    --address-prefix "0.0.0.0/0" \
    --next-hop-type VirtualAppliance \
    --next-hop-ip-address "$FW_IPV4"

# For IPv6, Azure Firewall currently uses IPv4 for management
# IPv6 rules are enforced but routing uses IPv4 next-hop
```

## Conclusion

Azure Firewall supports IPv6 network rules for filtering traffic in hub-spoke architectures. Create network rules with IPv6 source and destination addresses using `::/0` for any IPv6 or specific CIDR ranges. Azure Firewall Premium supports advanced IDPS for IPv6 traffic. Route spoke IPv6 traffic through the firewall using user-defined routes. Note that Azure Firewall's management plane uses IPv4, but the rules engine inspects and filters both IPv4 and IPv6 data plane traffic.
