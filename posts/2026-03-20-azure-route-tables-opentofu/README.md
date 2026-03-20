# How to Configure Azure Route Tables with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Route Tables, UDR, Network Routing, NVA, Infrastructure as Code

Description: Learn how to configure Azure Route Tables with OpenTofu to control network traffic flow within VNets, force tunneling through Network Virtual Appliances, and implement hub-spoke routing.

## Introduction

Azure Route Tables (User Defined Routes/UDRs) override Azure's default system routes to control how traffic is routed between subnets, to the internet, and to on-premises networks. They are essential for hub-spoke network architectures where you want to route all traffic through a central NVA (firewall, router), for forcing internet traffic through Azure Firewall, and for implementing split tunneling in VPN scenarios.

## Prerequisites

- OpenTofu v1.6+
- Azure credentials configured
- A Virtual Network with subnets

## Step 1: Create Route Table with Custom Routes

```hcl
resource "azurerm_route_table" "app" {
  name                = "${var.project_name}-app-rt"
  location            = var.location
  resource_group_name = var.resource_group_name

  # Disable BGP route propagation (prevents on-prem routes from overriding UDRs)
  disable_bgp_route_propagation = true

  route {
    name                   = "to-hub-firewall"
    address_prefix         = "0.0.0.0/0"  # Default route
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = var.firewall_private_ip
  }

  route {
    name           = "to-internet-direct"
    address_prefix = "8.8.8.8/32"  # Allow direct DNS (bypass firewall)
    next_hop_type  = "Internet"
  }

  tags = {
    Name = "${var.project_name}-app-route-table"
  }
}

# Associate route table with subnet

resource "azurerm_subnet_route_table_association" "app" {
  subnet_id      = var.app_subnet_id
  route_table_id = azurerm_route_table.app.id
}
```

## Step 2: Force Tunnel Internet Traffic Through Azure Firewall

```hcl
resource "azurerm_route_table" "spoke_subnets" {
  name                          = "${var.project_name}-spoke-rt"
  location                      = var.location
  resource_group_name           = var.resource_group_name
  disable_bgp_route_propagation = true

  # Force all internet traffic through Azure Firewall in hub
  route {
    name                   = "force-internet-through-firewall"
    address_prefix         = "0.0.0.0/0"
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = var.azure_firewall_private_ip
  }

  # Route to other spokes via firewall
  route {
    name                   = "to-spoke-2"
    address_prefix         = var.spoke_2_cidr
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = var.azure_firewall_private_ip
  }

  # Route to on-premises via VPN/ExpressRoute gateway (bypass firewall for on-prem)
  route {
    name           = "to-on-prem"
    address_prefix = "192.168.0.0/16"
    next_hop_type  = "VirtualNetworkGateway"
  }
}

# Apply to all spoke subnets
resource "azurerm_subnet_route_table_association" "spoke" {
  for_each = toset(var.spoke_subnet_ids)

  subnet_id      = each.value
  route_table_id = azurerm_route_table.spoke_subnets.id
}
```

## Step 3: Route Table for AKS Nodes (Kubenet)

```hcl
# Required when AKS uses kubenet networking
resource "azurerm_route_table" "aks" {
  name                          = "${var.project_name}-aks-rt"
  location                      = var.location
  resource_group_name           = var.resource_group_name
  disable_bgp_route_propagation = false  # Allow BGP for AKS

  # AKS adds pod CIDR routes automatically during cluster creation
  tags = {
    Name = "${var.project_name}-aks-route-table"
  }
}

resource "azurerm_subnet_route_table_association" "aks" {
  subnet_id      = azurerm_subnet.aks.id
  route_table_id = azurerm_route_table.aks.id
}
```

## Step 4: Multi-Region Route Tables

```hcl
variable "regions" {
  type = map(object({
    location             = string
    subnet_ids           = list(string)
    firewall_private_ip  = string
    resource_group_name  = string
  }))
}

resource "azurerm_route_table" "regional" {
  for_each = var.regions

  name                          = "${var.project_name}-${each.key}-rt"
  location                      = each.value.location
  resource_group_name           = each.value.resource_group_name
  disable_bgp_route_propagation = true

  route {
    name                   = "default-to-firewall"
    address_prefix         = "0.0.0.0/0"
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = each.value.firewall_private_ip
  }
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# Check effective routes on a NIC (shows system + UDR routes)
az network nic show-effective-route-table \
  --resource-group <rg> \
  --name <nic-name> \
  --output table

# Troubleshoot routing with IP flow verify
az network watcher test-ip-flow \
  --vm <vm-id> \
  --direction Outbound \
  --protocol TCP \
  --local <private-ip>:0 \
  --remote 1.1.1.1:443
```

## Conclusion

Set `disable_bgp_route_propagation = true` in spoke subnets to prevent VPN/ExpressRoute BGP routes from overriding your UDRs-this is critical when routing all traffic through Azure Firewall because an on-prem BGP route could create a routing loop. Use `next_hop_type = "VirtualNetworkGateway"` (not a specific IP) for on-premises routes so the traffic takes the active gateway without hardcoding the gateway IP. AKS with kubenet networking requires a pre-existing route table on the node subnet-Azure needs write permissions to the route table to add pod CIDR routes during node pool operations.
