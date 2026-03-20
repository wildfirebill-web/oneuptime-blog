# How to Configure Subnet-Level Peering in Azure VNet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, VNet, Peering, IPv4, Networking, Cloud

Description: Set up Azure Virtual Network peering between VNets to enable private IPv4 communication between subnets in different VNets without traversing the public internet.

## Introduction

Azure VNet peering connects two virtual networks so resources in each can communicate using private IPv4 addresses, as if they were on the same network. Peering works within the same region (VNet peering) or across regions (Global VNet peering) and supports both Azure-to-Azure and hub-and-spoke topologies.

## Prerequisites

- Two Azure VNets with non-overlapping IPv4 address spaces
- The VNets must be in the same Azure AD tenant (or use cross-tenant peering)
- Contributor role on both VNets

## VNet Address Space Planning

Peered VNets must not have overlapping CIDR blocks:

```
VNet A: 10.1.0.0/16
  - Subnet A1: 10.1.1.0/24
  - Subnet A2: 10.1.2.0/24

VNet B: 10.2.0.0/16
  - Subnet B1: 10.2.1.0/24
```

## Creating VNet Peering with Azure CLI

Peering is bidirectional — you must create a peering link from A to B AND from B to A:

```bash
# Peer VNet A to VNet B
az network vnet peering create \
  --name "vnetA-to-vnetB" \
  --resource-group rg-network \
  --vnet-name vnet-a \
  --remote-vnet /subscriptions/SUB_ID/resourceGroups/rg-network/providers/Microsoft.Network/virtualNetworks/vnet-b \
  --allow-vnet-access true \
  --allow-forwarded-traffic true

# Peer VNet B to VNet A (required for bidirectional communication)
az network vnet peering create \
  --name "vnetB-to-vnetA" \
  --resource-group rg-network \
  --vnet-name vnet-b \
  --remote-vnet /subscriptions/SUB_ID/resourceGroups/rg-network/providers/Microsoft.Network/virtualNetworks/vnet-a \
  --allow-vnet-access true \
  --allow-forwarded-traffic true
```

## Verifying Peering Status

```bash
# Check peering state (should be "Connected")
az network vnet peering show \
  --name "vnetA-to-vnetB" \
  --resource-group rg-network \
  --vnet-name vnet-a \
  --query "{State:peeringState, RemoteVNet:remoteVirtualNetwork.id}"
```

## Allowing Gateway Transit

For hub-and-spoke, allow spokes to use the hub's VPN or ExpressRoute gateway:

```bash
# On the hub peering (allow gateway transit)
az network vnet peering update \
  --name "hub-to-spoke" \
  --resource-group rg-network \
  --vnet-name hub-vnet \
  --set allowGatewayTransit=true

# On the spoke peering (use remote gateways)
az network vnet peering update \
  --name "spoke-to-hub" \
  --resource-group rg-network \
  --vnet-name spoke-vnet \
  --set useRemoteGateways=true
```

## Terraform Configuration

```hcl
resource "azurerm_virtual_network_peering" "a_to_b" {
  name                      = "vnetA-to-vnetB"
  resource_group_name       = azurerm_resource_group.net.name
  virtual_network_name      = azurerm_virtual_network.a.name
  remote_virtual_network_id = azurerm_virtual_network.b.id

  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
}

resource "azurerm_virtual_network_peering" "b_to_a" {
  name                      = "vnetB-to-vnetA"
  resource_group_name       = azurerm_resource_group.net.name
  virtual_network_name      = azurerm_virtual_network.b.name
  remote_virtual_network_id = azurerm_virtual_network.a.id

  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
}
```

## Network Security Groups and Peering

Peering creates a route between VNets, but NSG rules still apply. Ensure NSGs on target subnets allow traffic from the peered VNet's address space:

```bash
# Allow traffic from VNet A's range to VNet B's subnet
az network nsg rule create \
  --resource-group rg-network \
  --nsg-name nsg-subnet-b1 \
  --name allow-vnetA \
  --priority 200 \
  --source-address-prefixes 10.1.0.0/16 \
  --destination-port-ranges '*' \
  --access Allow
```

## Conclusion

Azure VNet peering is the lowest-latency, highest-throughput way to connect Azure VNets. It uses the Azure backbone network and avoids the public internet entirely, making it ideal for connecting production environments, management networks, and hub-and-spoke topologies.
