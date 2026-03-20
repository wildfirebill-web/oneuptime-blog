# How to Configure User-Defined Routes for IPv4 in Azure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, UDR, User-Defined Routes, IPv4, Routing, Route Tables

Description: Create Azure User-Defined Routes (route tables) to override system routes and force traffic through network virtual appliances, firewalls, or specific next hops.

## Introduction

Azure automatically creates system routes for VNet traffic. User-Defined Routes (UDRs) override these defaults. Common use cases: forcing internet traffic through a firewall NVA, directing traffic between subnets through an inspection appliance, or overriding BGP routes learned from VPN/ExpressRoute.

## Creating a Route Table

```bash
RESOURCE_GROUP="my-network-rg"

az network route-table create \
  --resource-group $RESOURCE_GROUP \
  --name app-route-table \
  --location eastus \
  --disable-bgp-route-propagation false
```

`--disable-bgp-route-propagation false` (default) allows VPN gateway-learned BGP routes to appear in this table. Set to `true` to prevent BGP routes.

## Adding Routes

```bash
# Force all internet traffic through a Network Virtual Appliance (NVA)
az network route-table route create \
  --resource-group $RESOURCE_GROUP \
  --route-table-name app-route-table \
  --name force-internet-via-nva \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.100.5.4

# Route to on-premises via VPN gateway
az network route-table route create \
  --resource-group $RESOURCE_GROUP \
  --route-table-name app-route-table \
  --name to-onprem \
  --address-prefix 192.168.0.0/16 \
  --next-hop-type VirtualNetworkGateway

# Blackhole a specific subnet (drop traffic)
az network route-table route create \
  --resource-group $RESOURCE_GROUP \
  --route-table-name app-route-table \
  --name blackhole-mgmt \
  --address-prefix 10.100.99.0/24 \
  --next-hop-type None
```

## Next Hop Types

| Next Hop Type | Description |
|---|---|
| VirtualNetworkGateway | Route to VPN/ExpressRoute gateway |
| VnetLocal | Route within the VNet |
| Internet | Route to internet |
| VirtualAppliance | Route to NVA (requires IP) |
| None | Drop (blackhole) |

## Associating the Route Table with a Subnet

```bash
# Apply the route table to the app subnet
az network vnet subnet update \
  --resource-group $RESOURCE_GROUP \
  --vnet-name prod-vnet \
  --name app-subnet \
  --route-table app-route-table
```

## Verifying Effective Routes on a VM NIC

```bash
# Check effective routes (system + UDR + BGP combined)
az network nic show-effective-route-table \
  --resource-group $RESOURCE_GROUP \
  --name app-vm-nic \
  --output table
```

This shows which routes are actually active on a NIC, including their source (System, User, BGP).

## Listing Routes in a Route Table

```bash
az network route-table route list \
  --resource-group $RESOURCE_GROUP \
  --route-table-name app-route-table \
  --output table
```

## Removing a Route

```bash
az network route-table route delete \
  --resource-group $RESOURCE_GROUP \
  --route-table-name app-route-table \
  --name force-internet-via-nva
```

## Disassociating a Route Table

```bash
az network vnet subnet update \
  --resource-group $RESOURCE_GROUP \
  --vnet-name prod-vnet \
  --name app-subnet \
  --remove routeTable
```

## Conclusion

Create a route table with `az network route-table create`, add routes with `VirtualAppliance`, `VirtualNetworkGateway`, `Internet`, or `None` next hops, then associate it with subnets. Use `az network nic show-effective-route-table` to see which routes are actually applied to a VM — this is the first tool to reach for when debugging routing issues.
