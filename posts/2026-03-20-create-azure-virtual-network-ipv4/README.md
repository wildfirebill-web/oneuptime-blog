# How to Create an Azure Virtual Network with an IPv4 Address Space

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, VNet, IPv4, Networking, Cloud, Virtual Network

Description: Create an Azure Virtual Network with a custom IPv4 address space using the Azure CLI, Portal, or ARM templates, and understand address space planning fundamentals.

## Introduction

An Azure Virtual Network (VNet) is the fundamental building block of private networking in Azure. Every VM, load balancer, and private endpoint lives inside a VNet. You define the IPv4 address space at creation time, so planning CIDR blocks before deployment is essential.

## Prerequisites

```bash
# Install Azure CLI and login

az login
az account set --subscription "your-subscription-id"

# Create a resource group
az group create \
  --name my-network-rg \
  --location eastus
```

## Creating a VNet with Azure CLI

```bash
# Create a VNet with a /16 address space
az network vnet create \
  --resource-group my-network-rg \
  --name my-vnet \
  --address-prefix 10.0.0.0/16 \
  --location eastus

# Verify the creation
az network vnet show \
  --resource-group my-network-rg \
  --name my-vnet \
  --query '{name:name, addressSpace:addressSpace.addressPrefixes}' \
  --output table
```

## Adding Multiple Address Spaces

A VNet can have multiple non-overlapping address prefixes:

```bash
# Add a secondary address space to an existing VNet
az network vnet update \
  --resource-group my-network-rg \
  --name my-vnet \
  --add addressSpace.addressPrefixes "172.16.0.0/24"

# View all address spaces
az network vnet show \
  --resource-group my-network-rg \
  --name my-vnet \
  --query 'addressSpace.addressPrefixes'
```

## Creating a VNet with a Subnet in One Command

```bash
az network vnet create \
  --resource-group my-network-rg \
  --name prod-vnet \
  --address-prefix 10.100.0.0/16 \
  --subnet-name default-subnet \
  --subnet-prefix 10.100.0.0/24 \
  --location eastus
```

## ARM Template for VNet Creation

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Network/virtualNetworks",
      "apiVersion": "2023-05-01",
      "name": "prod-vnet",
      "location": "[resourceGroup().location]",
      "properties": {
        "addressSpace": {
          "addressPrefixes": ["10.100.0.0/16"]
        },
        "subnets": [
          {
            "name": "web-subnet",
            "properties": {
              "addressPrefix": "10.100.1.0/24"
            }
          },
          {
            "name": "app-subnet",
            "properties": {
              "addressPrefix": "10.100.2.0/24"
            }
          }
        ]
      }
    }
  ]
}
```

## IPv4 Address Space Planning

| Tier | CIDR | Hosts | Use Case |
|---|---|---|---|
| VNet | 10.0.0.0/16 | 65,536 | Overall network |
| Web tier | 10.0.1.0/24 | 251 | Public-facing VMs |
| App tier | 10.0.2.0/24 | 251 | Application servers |
| Data tier | 10.0.3.0/24 | 251 | Databases |
| Gateway | 10.0.255.0/27 | 27 | VPN/ExpressRoute |

Note: Azure reserves 5 IP addresses per subnet (first 4 and last 1).

## DNS Configuration for the VNet

```bash
# Set custom DNS servers for the VNet
az network vnet update \
  --resource-group my-network-rg \
  --name my-vnet \
  --dns-servers 10.0.1.4 10.0.1.5

# Use Azure's default DNS (clear custom DNS)
az network vnet update \
  --resource-group my-network-rg \
  --name my-vnet \
  --dns-servers ""
```

## Listing VNets in a Subscription

```bash
# List all VNets in the resource group
az network vnet list \
  --resource-group my-network-rg \
  --output table

# List all VNets across all resource groups
az network vnet list --output table
```

## Conclusion

Create Azure VNets with the `az network vnet create` command specifying `--address-prefix` with a CIDR block. Plan the address space ahead of time to avoid overlaps with on-premises networks, other VNets, and future peered networks. Azure reserves 5 addresses per subnet, so account for this when sizing subnets.
