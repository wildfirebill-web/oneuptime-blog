# How to Configure Private Endpoint for Azure Cache for Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Azure Cache for Redis, Private Endpoint, Azure Networking, Security

Description: Learn how to configure Azure Private Endpoint for Azure Cache for Redis to restrict access to your VNet and eliminate public internet exposure.

---

## Why Use Private Endpoints

By default, Azure Cache for Redis is accessible over the public internet (restricted by firewall rules). Private Endpoint creates a network interface with a private IP in your VNet, so traffic from your applications to Redis never leaves the Azure backbone network.

Benefits:
- Redis is not exposed to the internet at all
- Traffic stays within Azure's private network
- Required for many compliance frameworks (PCI-DSS, HIPAA)
- Works across VNets via VNet peering or Private DNS

Private endpoints are available on all Azure Cache for Redis tiers (Basic, Standard, Premium, Enterprise).

## Architecture

```text
Your VNet (10.0.0.0/16)
  App Subnet (10.0.1.0/24)
    App Service / AKS / VM
         |
         | private traffic via private IP
         |
  Private Endpoint Subnet (10.0.2.0/24)
    Private Endpoint NIC: 10.0.2.5
         |
         | Azure backbone (never public internet)
         |
Azure Cache for Redis
(public endpoint disabled)
```

## Creating a Private Endpoint via Azure CLI

```bash
# Step 1: Create the Private Endpoint
az network private-endpoint create \
  --name redis-private-endpoint \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet private-endpoints-subnet \
  --private-connection-resource-id $(az redis show \
    --name my-redis-cache \
    --resource-group myRG \
    --query id \
    --output tsv) \
  --group-id redisCache \
  --connection-name my-redis-connection

# Step 2: Create Private DNS Zone for Redis
az network private-dns zone create \
  --resource-group myRG \
  --name "privatelink.redis.cache.windows.net"

# Step 3: Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name "privatelink.redis.cache.windows.net" \
  --name myDNSLink \
  --virtual-network myVNet \
  --registration-enabled false

# Step 4: Create DNS zone group to auto-register private endpoint IP
az network private-endpoint dns-zone-group create \
  --resource-group myRG \
  --endpoint-name redis-private-endpoint \
  --name redis-dns-zone-group \
  --private-dns-zone "privatelink.redis.cache.windows.net" \
  --zone-name redis
```

## Step 5: Disable Public Network Access

```bash
# Disable public access once private endpoint is working
az redis update \
  --name my-redis-cache \
  --resource-group myRG \
  --set publicNetworkAccess=Disabled
```

## Terraform Configuration

```hcl
# Subnet for private endpoints (disable private endpoint network policies)
resource "azurerm_subnet" "private_endpoints" {
  name                 = "private-endpoints"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/24"]

  # Required for private endpoints
  private_endpoint_network_policies = "Disabled"
}

# Azure Cache for Redis (public access will be disabled)
resource "azurerm_redis_cache" "private" {
  name                          = "my-private-redis"
  location                      = azurerm_resource_group.main.location
  resource_group_name           = azurerm_resource_group.main.name
  capacity                      = 2
  family                        = "C"
  sku_name                      = "Standard"

  # Disable public network access
  public_network_access_enabled = false

  redis_configuration {
    maxmemory_policy = "allkeys-lru"
  }
}

# Private Endpoint
resource "azurerm_private_endpoint" "redis" {
  name                = "redis-private-endpoint"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = azurerm_subnet.private_endpoints.id

  private_service_connection {
    name                           = "redis-connection"
    private_connection_resource_id = azurerm_redis_cache.private.id
    subresource_names              = ["redisCache"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "redis-dns-zone-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.redis.id]
  }
}

# Private DNS Zone
resource "azurerm_private_dns_zone" "redis" {
  name                = "privatelink.redis.cache.windows.net"
  resource_group_name = azurerm_resource_group.main.name
}

# Link DNS zone to VNet
resource "azurerm_private_dns_zone_virtual_network_link" "redis" {
  name                  = "redis-dns-vnet-link"
  resource_group_name   = azurerm_resource_group.main.name
  private_dns_zone_name = azurerm_private_dns_zone.redis.name
  virtual_network_id    = azurerm_virtual_network.main.id
  registration_enabled  = false
}
```

## Verifying Private Endpoint Works

```bash
# From a VM or container in your VNet, verify DNS resolves to private IP
nslookup my-redis-cache.redis.cache.windows.net
# Should return: 10.0.2.5 (private IP, not a public IP)

# If it returns a public IP, DNS zone is not linked correctly

# Test connectivity
redis-cli \
  -h my-redis-cache.redis.cache.windows.net \
  -p 6380 \
  --tls \
  -a "your-access-key" \
  ping
```

Expected DNS output:

```text
Server:    168.63.129.16
Address:   168.63.129.16#53

Non-authoritative answer:
my-redis-cache.redis.cache.windows.net
  canonical name = my-redis-cache.privatelink.redis.cache.windows.net.
Name: my-redis-cache.privatelink.redis.cache.windows.net
Address: 10.0.2.5
```

## Cross-VNet Access via VNet Peering

```bash
# Peer app VNet with the VNet containing the private endpoint
az network vnet peering create \
  --name AppToRedisVNet \
  --resource-group appRG \
  --vnet-name appVNet \
  --remote-vnet /subscriptions/.../resourceGroups/redisRG/providers/Microsoft.Network/virtualNetworks/redisVNet \
  --allow-vnet-access

# Also link the Private DNS Zone to the app VNet
az network private-dns link vnet create \
  --resource-group redisRG \
  --zone-name "privatelink.redis.cache.windows.net" \
  --name appVNetLink \
  --virtual-network /subscriptions/.../resourceGroups/appRG/providers/Microsoft.Network/virtualNetworks/appVNet \
  --registration-enabled false
```

## Troubleshooting

```text
Issue: DNS still resolves to public IP
Fix: Verify Private DNS Zone is linked to the VNet resolving the name.
     Each VNet that needs private access needs its own DNS link.

Issue: Connection refused to private IP
Fix: Check the private endpoint NIC shows "Approved" in the portal.
     Verify NSG on the private endpoint subnet allows port 6379/6380.

Issue: Private endpoint shows "Disconnected"
Fix: Re-approve the connection:
     az network private-endpoint-connection approve
       --name redis-connection
       --resource-group myRG
       --resource-name my-redis-cache
       --type Microsoft.Cache/Redis
```

## Summary

Azure Private Endpoint for Redis creates a private IP in your VNet that routes all Redis traffic through Azure's internal network, eliminating public internet exposure. The setup requires creating the private endpoint, configuring a private DNS zone linked to your VNet, and disabling public network access on the cache. For multi-VNet environments, peer the VNets and link the private DNS zone to each VNet that needs access to Redis.
