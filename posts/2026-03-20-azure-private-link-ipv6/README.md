# How to Configure Azure Private Link IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, Private Link, IPv6, Private Endpoint, Dual-Stack, Services

Description: Configure Azure Private Link endpoints with IPv6 addresses for private connectivity to Azure PaaS services.

## Introduction

Azure Private Link IPv6 enables private IPv6 connectivity between cloud resources and on-premises or inter-VPC networks. Proper configuration requires setting up dual-stack support, IPv6 BGP sessions, and route advertisement.

## Prerequisites

- VPC/VNet with dual-stack (IPv4 + IPv6) subnets
- An existing Azure account with appropriate IAM permissions
- IPv6 address space allocated for the connection

## Step 1: Verify IPv6 Prerequisites

```bash
# Check VPC has IPv6 CIDR

az network vnet show --resource-group myRG --name myVNet --query 'addressSpace'
```

## Step 2: Enable IPv6 on the Service

```bash
# Verify VNet has IPv6 address space
az network vnet show     --resource-group myRG     --name myVNet     --query "addressSpace.addressPrefixes"

# Add IPv6 address space if missing
az network vnet update     --resource-group myRG     --name myVNet     --add addressSpace.addressPrefixes "2001:db8::/48"
```

## Step 3: Configure IPv6 BGP

```bash
# Create ExpressRoute circuit with IPv6
az network express-route create     --resource-group myRG     --name myCircuit     --location eastus     --provider "Equinix"     --peering-location "Silicon Valley"     --bandwidth 1000     --sku-family MeteredData     --sku-tier Standard

# Configure IPv6 private peering
az network express-route peering create     --resource-group myRG     --circuit-name myCircuit     --peering-type AzurePrivatePeering     --peer-asn 65000     --primary-peer-address-prefix "2001:db8:primary::/126"     --secondary-peer-address-prefix "2001:db8:secondary::/126"
```

## Step 4: Add IPv6 Routes

```bash
# Add IPv6 static route
az network route-table route create \
    --resource-group myRG \
    --route-table-name myRT \
    --name ipv6-default \
    --address-prefix '::/0' \
    --next-hop-type VirtualNetworkGateway
```

## Step 5: Test IPv6 Connectivity

```bash
# Test from cloud instance
ping6 -c 3 <on-premises-ipv6-address>

# Verify route is learned
az network nic show-effective-route-table --resource-group myRG --name myNIC | grep -A2 'ipv6'
```

## Step 6: Terraform Example

```hcl
# Terraform for Azure Private Link IPv6
resource "azurerm_express_route_circuit_peering" "ipv6" {
  peering_type                  = "AzurePrivatePeering"
  express_route_circuit_name    = azurerm_express_route_circuit.main.name
  resource_group_name           = azurerm_resource_group.main.name
  peer_asn                      = 65000
  primary_peer_address_prefix   = "2001:db8:primary::/126"
  secondary_peer_address_prefix = "2001:db8:secondary::/126"
  vlan_id                       = 100
}
```

## Conclusion

Azure Private Link IPv6 requires enabling dual-stack at the subnet level, configuring IPv6 BGP sessions, and adding IPv6 routes in the relevant route tables. Test connectivity end-to-end after configuration. Use Terraform for declarative, repeatable deployments. Monitor IPv6 BGP session state and route advertisement with OneUptime's network health checks.
