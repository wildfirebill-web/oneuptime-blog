# How to Configure Private Endpoints for IPv4 in Azure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, Private Endpoint, IPv4, Private Link, Networking, Security

Description: Create Azure Private Endpoints to access PaaS services like Azure Storage and SQL Database over private IPv4 addresses within your VNet, eliminating public internet exposure.

## Introduction

Azure Private Endpoints assign a private IPv4 address from your VNet's address space to an Azure PaaS resource. Traffic to the service flows entirely over Microsoft's private network. This prevents data exfiltration via the public internet and allows NSGs to restrict access to the private endpoint.

## Creating a Private Endpoint for Azure Storage

```bash
RESOURCE_GROUP="my-network-rg"
STORAGE_ACCOUNT="mystorageaccount"

# Get the storage account resource ID
STORAGE_ID=$(az storage account show \
  --resource-group $RESOURCE_GROUP \
  --name $STORAGE_ACCOUNT \
  --query id --output tsv)

# Disable network policies on the target subnet (required for private endpoints)
az network vnet subnet update \
  --resource-group $RESOURCE_GROUP \
  --vnet-name prod-vnet \
  --name pe-subnet \
  --disable-private-endpoint-network-policies true

# Create the private endpoint for blob storage
az network private-endpoint create \
  --resource-group $RESOURCE_GROUP \
  --name storage-pe \
  --vnet-name prod-vnet \
  --subnet pe-subnet \
  --private-connection-resource-id $STORAGE_ID \
  --group-id blob \
  --connection-name storage-pe-conn \
  --location eastus
```

## Creating a Private Endpoint for Azure SQL

```bash
SQL_SERVER_ID=$(az sql server show \
  --resource-group $RESOURCE_GROUP \
  --name my-sql-server \
  --query id --output tsv)

az network private-endpoint create \
  --resource-group $RESOURCE_GROUP \
  --name sql-pe \
  --vnet-name prod-vnet \
  --subnet pe-subnet \
  --private-connection-resource-id $SQL_SERVER_ID \
  --group-id sqlServer \
  --connection-name sql-pe-conn \
  --location eastus
```

## Setting Up Private DNS Zone Integration

Without DNS integration, names resolve to public IPs. Create a private DNS zone that overrides resolution within your VNet:

```bash
# Create private DNS zone for blob storage
az network private-dns zone create \
  --resource-group $RESOURCE_GROUP \
  --name privatelink.blob.core.windows.net

# Link the DNS zone to the VNet
az network private-dns link vnet create \
  --resource-group $RESOURCE_GROUP \
  --zone-name privatelink.blob.core.windows.net \
  --name vnet-dns-link \
  --virtual-network prod-vnet \
  --registration-enabled false

# Create DNS zone group on the private endpoint
az network private-endpoint dns-zone-group create \
  --resource-group $RESOURCE_GROUP \
  --endpoint-name storage-pe \
  --name storage-dns-group \
  --private-dns-zone privatelink.blob.core.windows.net \
  --zone-name blob
```

## Verifying the Private Endpoint IP

```bash
# Show the private IP assigned to the endpoint
az network private-endpoint show \
  --resource-group $RESOURCE_GROUP \
  --name storage-pe \
  --query 'customDnsConfigs[].{FQDN:fqdn, IP:ipAddresses[0]}' \
  --output table
```

## Testing Private Endpoint Connectivity

```bash
# From a VM in the VNet, test resolution
nslookup mystorageaccount.blob.core.windows.net
# Should return a 10.x.x.x address, not 52.x.x.x

# Test connectivity
curl https://mystorageaccount.blob.core.windows.net -v 2>&1 | grep "Connected to"
```

## Approved vs Pending Connections

If the storage account is in another subscription, the owner must approve:

```bash
# List pending connections
az network private-endpoint-connection list \
  --id $STORAGE_ID \
  --query '[?properties.privateLinkServiceConnectionState.status==`Pending`]'

# Approve a pending connection
az network private-endpoint-connection approve \
  --resource-group $RESOURCE_GROUP \
  --resource-name $STORAGE_ACCOUNT \
  --name connection-name \
  --type Microsoft.Storage/storageAccounts \
  --description "Approved"
```

## Conclusion

Private Endpoints assign private IPv4 addresses to Azure PaaS services. Always create a private DNS zone and link it to your VNet so that service FQDNs resolve to private IPs. Disable `PrivateEndpointNetworkPolicies` on the subnet, then create the endpoint with the correct `--group-id` for the service subresource.
