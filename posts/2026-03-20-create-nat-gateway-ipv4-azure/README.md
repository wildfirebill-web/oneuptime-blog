# How to Create a NAT Gateway for IPv4 Outbound Connectivity in Azure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, NAT Gateway, IPv4, Outbound Connectivity, Networking

Description: Create an Azure NAT Gateway and associate it with a subnet to provide reliable, scalable outbound IPv4 connectivity for private VMs without public IP addresses.

## Introduction

Azure NAT Gateway provides outbound-only IPv4 connectivity for VMs in private subnets. Unlike using a load balancer's outbound rules or individual public IPs, NAT Gateway scales automatically up to 64,512 SNAT ports per public IP, eliminating SNAT port exhaustion for high-concurrency workloads.

## Step 1: Create a Public IP for the NAT Gateway

```bash
RESOURCE_GROUP="my-network-rg"

# Create a static public IP for outbound traffic

az network public-ip create \
  --resource-group $RESOURCE_GROUP \
  --name nat-pip \
  --sku Standard \
  --allocation-method Static \
  --location eastus
```

## Step 2: Create the NAT Gateway

```bash
# Create the NAT gateway with a 4-minute idle timeout
az network nat gateway create \
  --resource-group $RESOURCE_GROUP \
  --name my-nat-gateway \
  --public-ip-addresses nat-pip \
  --idle-timeout 4 \
  --location eastus
```

## Step 3: Associate the NAT Gateway with a Subnet

```bash
# Attach to the private app subnet
az network vnet subnet update \
  --resource-group $RESOURCE_GROUP \
  --vnet-name prod-vnet \
  --name app-subnet \
  --nat-gateway my-nat-gateway
```

After this, all outbound IPv4 traffic from `app-subnet` uses the NAT Gateway's public IP.

## Verifying NAT Gateway is Active

```bash
# Show NAT gateway configuration
az network nat gateway show \
  --resource-group $RESOURCE_GROUP \
  --name my-nat-gateway \
  --query '{name:name, publicIPs:publicIpAddresses[].id, provisioningState:provisioningState}' \
  --output json

# Check the subnet's NAT gateway association
az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name prod-vnet \
  --name app-subnet \
  --query 'natGateway.id' \
  --output tsv
```

## Using a Public IP Prefix for Multiple Outbound IPs

A public IP prefix provides a contiguous block of public IPs, useful for allowing known SNAT addresses:

```bash
# Create a /31 public IP prefix (2 public IPs)
az network public-ip prefix create \
  --resource-group $RESOURCE_GROUP \
  --name nat-pip-prefix \
  --prefix-length 31 \
  --location eastus

# Create NAT gateway using the prefix
az network nat gateway create \
  --resource-group $RESOURCE_GROUP \
  --name my-nat-gateway \
  --public-ip-prefixes nat-pip-prefix \
  --idle-timeout 4 \
  --location eastus
```

## Idle Timeout Configuration

| Timeout | Use Case |
|---|---|
| 4 minutes | Default, suits most workloads |
| 30 minutes | Long-lived connections (database, SSH) |

```bash
# Update idle timeout
az network nat gateway update \
  --resource-group $RESOURCE_GROUP \
  --name my-nat-gateway \
  --idle-timeout 30
```

## Removing NAT Gateway from a Subnet

```bash
az network vnet subnet update \
  --resource-group $RESOURCE_GROUP \
  --vnet-name prod-vnet \
  --name app-subnet \
  --remove natGateway
```

## NAT Gateway vs Other Outbound Methods

| Method | SNAT Ports | Scale | Best For |
|---|---|---|---|
| NAT Gateway | 64,512/IP | Automatic | High concurrency |
| Load Balancer outbound | 1,024–64,512 | Manual | Load-balanced VMs |
| Instance Public IP | 64,512 | Per VM | Single VMs |

## Conclusion

Create an Azure NAT Gateway with a Standard SKU public IP, then associate it with private subnets using `az network vnet subnet update --nat-gateway`. NAT Gateway is preferred over load balancer outbound rules for workloads requiring high SNAT port counts or predictable outbound IP addresses.
