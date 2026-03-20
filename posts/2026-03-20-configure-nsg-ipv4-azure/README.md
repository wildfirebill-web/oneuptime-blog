# How to Configure Network Security Groups for IPv4 Traffic in Azure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, NSG, Network Security Group, IPv4, Firewall, Security

Description: Create and configure Azure Network Security Groups with inbound and outbound rules to control IPv4 traffic for subnets and virtual machine network interfaces.

## Introduction

Network Security Groups (NSGs) are Azure's stateful packet filters. Each NSG contains inbound and outbound security rules with priority numbers (100–4096, lower = higher priority). Associate NSGs with subnets or individual VM NICs. Traffic matching a rule is either allowed or denied; traffic with no matching rule hits the default deny-all.

## Creating an NSG

```bash
RESOURCE_GROUP="my-network-rg"

# Create an NSG

az network nsg create \
  --resource-group $RESOURCE_GROUP \
  --name web-nsg \
  --location eastus
```

## Adding Inbound Rules

```bash
# Allow HTTP from anywhere
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name web-nsg \
  --name allow-http \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes '*' \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 80

# Allow HTTPS from anywhere
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name web-nsg \
  --name allow-https \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes '*' \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 443

# Allow SSH from specific IP only
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name web-nsg \
  --name allow-ssh-admin \
  --priority 200 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 203.0.113.10/32 \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 22
```

## Adding Outbound Rules

```bash
# Deny outbound to an internal blocked subnet
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name web-nsg \
  --name deny-to-db \
  --priority 300 \
  --direction Outbound \
  --access Deny \
  --protocol '*' \
  --source-address-prefixes '*' \
  --source-port-ranges '*' \
  --destination-address-prefixes 10.100.3.0/24 \
  --destination-port-ranges '*'
```

## Associating an NSG with a Subnet

```bash
# Associate the NSG with the web subnet
az network vnet subnet update \
  --resource-group $RESOURCE_GROUP \
  --vnet-name prod-vnet \
  --name web-subnet \
  --network-security-group web-nsg
```

## Associating an NSG with a VM NIC

```bash
# Associate with a specific NIC
az network nic update \
  --resource-group $RESOURCE_GROUP \
  --name my-vm-nic \
  --network-security-group web-nsg
```

## Listing NSG Rules

```bash
# List all rules in an NSG
az network nsg rule list \
  --resource-group $RESOURCE_GROUP \
  --nsg-name web-nsg \
  --query '[].{Name:name, Priority:priority, Direction:direction, Access:access, Protocol:protocol, Dest:destinationPortRanges}' \
  --output table
```

## Default NSG Rules

Every NSG includes these non-deletable default rules:

| Priority | Name | Direction | Access | Description |
|---|---|---|---|---|
| 65000 | AllowVnetInBound | Inbound | Allow | VNet to VNet |
| 65001 | AllowAzureLoadBalancerInBound | Inbound | Allow | Azure LB probes |
| 65500 | DenyAllInBound | Inbound | Deny | Deny all |
| 65000 | AllowVnetOutBound | Outbound | Allow | VNet to VNet |
| 65001 | AllowInternetOutBound | Outbound | Allow | Internet access |
| 65500 | DenyAllOutBound | Outbound | Deny | Deny all |

## Viewing Effective Security Rules

```bash
# View effective rules on a VM NIC (merged subnet + NIC NSGs)
az network nic list-effective-nsg \
  --resource-group $RESOURCE_GROUP \
  --name my-vm-nic
```

## Conclusion

Create NSGs with `az network nsg create`, add rules with `az network nsg rule create` using priority numbers and ALLOW/DENY actions, then associate with subnets or NICs. Lower priority numbers win. Always check effective security rules on a NIC to understand the combined subnet + NIC NSG view.
