# How to Configure Azure Load Balancer for IPv4 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, Load Balancer, IPv4, Networking, High Availability, Backend Pool

Description: Configure an Azure Standard Load Balancer with IPv4 frontend, backend pool, health probe, and load balancing rules to distribute traffic across virtual machines.

## Introduction

Azure Load Balancer operates at Layer 4 (TCP/UDP). The Standard SKU supports availability zones, high-port rules, and backend pools with any resource. A load balancer consists of a frontend IP, a backend pool, health probes, and load balancing rules.

## Step 1: Create a Public IP for the Frontend

```bash
RESOURCE_GROUP="my-network-rg"

az network public-ip create \
  --resource-group $RESOURCE_GROUP \
  --name lb-pip \
  --sku Standard \
  --allocation-method Static \
  --location eastus
```

## Step 2: Create the Load Balancer

```bash
az network lb create \
  --resource-group $RESOURCE_GROUP \
  --name my-lb \
  --sku Standard \
  --frontend-ip-name lb-frontend \
  --public-ip-address lb-pip \
  --backend-pool-name lb-backend \
  --location eastus
```

## Step 3: Create a Health Probe

```bash
# HTTP health probe on port 80

az network lb probe create \
  --resource-group $RESOURCE_GROUP \
  --lb-name my-lb \
  --name http-probe \
  --protocol Http \
  --port 80 \
  --path /health \
  --interval 15 \
  --threshold 2
```

## Step 4: Create a Load Balancing Rule

```bash
# Forward port 80 to backend pool port 80
az network lb rule create \
  --resource-group $RESOURCE_GROUP \
  --lb-name my-lb \
  --name http-rule \
  --frontend-ip-name lb-frontend \
  --frontend-port 80 \
  --backend-pool-name lb-backend \
  --backend-port 80 \
  --protocol Tcp \
  --probe-name http-probe \
  --idle-timeout 4
```

## Step 5: Add VMs to the Backend Pool

```bash
# Get NIC names for the backend VMs
NIC1="web-vm-01-nic"
NIC2="web-vm-02-nic"

# Add VM 1 to backend pool
az network nic ip-config update \
  --resource-group $RESOURCE_GROUP \
  --nic-name $NIC1 \
  --name ipconfig1 \
  --lb-name my-lb \
  --lb-address-pools lb-backend

# Add VM 2 to backend pool
az network nic ip-config update \
  --resource-group $RESOURCE_GROUP \
  --nic-name $NIC2 \
  --name ipconfig1 \
  --lb-name my-lb \
  --lb-address-pools lb-backend
```

## Creating an Internal (Private) Load Balancer

For internal traffic distribution:

```bash
az network lb create \
  --resource-group $RESOURCE_GROUP \
  --name internal-lb \
  --sku Standard \
  --frontend-ip-name lb-frontend \
  --private-ip-address 10.100.2.100 \
  --vnet-name prod-vnet \
  --subnet app-subnet \
  --backend-pool-name lb-backend \
  --location eastus
```

## Inbound NAT Rules for Direct VM Access

```bash
# Create NAT rule to reach VM 1 directly on port 50001
az network lb inbound-nat-rule create \
  --resource-group $RESOURCE_GROUP \
  --lb-name my-lb \
  --name ssh-to-vm1 \
  --frontend-ip-name lb-frontend \
  --frontend-port 50001 \
  --backend-port 22 \
  --protocol Tcp
```

## Viewing Load Balancer Health

```bash
# Show backend pool health
az network lb show \
  --resource-group $RESOURCE_GROUP \
  --name my-lb \
  --query 'backendAddressPools[0].backendIPConfigurations[].id'

# Get the frontend public IP
az network public-ip show \
  --resource-group $RESOURCE_GROUP \
  --name lb-pip \
  --query ipAddress \
  --output tsv
```

## Conclusion

Azure Load Balancer requires a frontend IP, backend pool, health probe, and load balancing rule. Use Standard SKU for zone-redundancy and more features. Add VMs via their NIC IP configurations to the backend pool. For private load balancing, omit the public IP and specify a private IP with subnet.
