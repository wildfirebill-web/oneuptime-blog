# How to Configure Azure Service Bus with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, IPv6, Service Bus, Messaging, Private Endpoint, Dual-Stack

Description: Configure Azure Service Bus to accept connections over IPv6 through Private Endpoints in dual-stack VNets and configure IP filtering rules for IPv6 client addresses.

## Introduction

Azure Service Bus supports IPv6 connectivity through Private Endpoints in dual-stack VNets. When a Private Endpoint is deployed in an IPv6-enabled subnet, clients can connect to Service Bus over IPv6. Additionally, Service Bus network rules support IPv6 address ranges for IP filtering. This enables message-based architectures to function in IPv6-only or dual-stack environments.

## Create Service Bus with Private Endpoint

```bash
RG="rg-servicebus-ipv6"
LOCATION="eastus"

# Create Service Bus namespace

az servicebus namespace create \
    --resource-group "$RG" \
    --name sb-myapp \
    --location "$LOCATION" \
    --sku Premium

# Get namespace resource ID
SB_ID=$(az servicebus namespace show \
    --resource-group "$RG" \
    --name sb-myapp \
    --query id --output tsv)

# Create Private Endpoint in dual-stack subnet
az network private-endpoint create \
    --resource-group "$RG" \
    --name pe-servicebus \
    --vnet-name vnet-dualstack \
    --subnet subnet-private \
    --private-connection-resource-id "$SB_ID" \
    --group-id namespace \
    --connection-name connection-servicebus

# Create private DNS zone for Service Bus
az network private-dns zone create \
    --resource-group "$RG" \
    --name "servicebus.windows.net"

az network private-dns link vnet create \
    --resource-group "$RG" \
    --zone-name "servicebus.windows.net" \
    --name link-vnet \
    --virtual-network vnet-dualstack \
    --registration-enabled false

# Add DNS record for the private endpoint
PRIVATE_IP=$(az network private-endpoint show \
    --resource-group "$RG" \
    --name pe-servicebus \
    --query "customDnsConfigs[0].ipAddresses[0]" \
    --output tsv)

az network private-dns record-set a add-record \
    --resource-group "$RG" \
    --zone-name "servicebus.windows.net" \
    --record-set-name "sb-myapp" \
    --ipv4-address "$PRIVATE_IP"
```

## Terraform Service Bus with IPv6

```hcl
# service_bus_ipv6.tf

resource "azurerm_servicebus_namespace" "main" {
  name                = "sb-myapp"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "Premium"

  # Enable public network access only for specific IPs
  public_network_access_enabled = false

  tags = { Name = "service-bus" }
}

# Private endpoint in dual-stack subnet
resource "azurerm_private_endpoint" "servicebus" {
  name                = "pe-servicebus"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = azurerm_subnet.private.id  # IPv6-enabled subnet

  private_service_connection {
    name                           = "conn-servicebus"
    private_connection_resource_id = azurerm_servicebus_namespace.main.id
    subresource_names              = ["namespace"]
    is_manual_connection           = false
  }

  # DNS zone group for automatic DNS configuration
  private_dns_zone_group {
    name = "dns-zone-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.servicebus.id]
  }
}

resource "azurerm_private_dns_zone" "servicebus" {
  name                = "privatelink.servicebus.windows.net"
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "servicebus" {
  name                  = "link-vnet"
  resource_group_name   = azurerm_resource_group.main.name
  private_dns_zone_name = azurerm_private_dns_zone.servicebus.name
  virtual_network_id    = azurerm_virtual_network.main.id
  registration_enabled  = false
}
```

## Add IPv6 Network Rules

```bash
# Add IPv6 CIDR to Service Bus network rules (public endpoint)
az servicebus namespace network-rule-set ip-rule add \
    --resource-group "$RG" \
    --namespace-name sb-myapp \
    --ip-mask "2001:db8:allowed::/48" \
    --action Allow

# List current network rules
az servicebus namespace network-rule-set show \
    --resource-group "$RG" \
    --namespace-name sb-myapp \
    --query "ipRules"
```

## Connect to Service Bus over IPv6

```python
from azure.servicebus import ServiceBusClient, ServiceBusMessage

# Connection over IPv6 via Private Endpoint
# The private endpoint DNS resolves to IPv6 when in dual-stack subnet
connection_str = "Endpoint=sb://sb-myapp.servicebus.windows.net/;..."

# Force IPv6 DNS resolution (Python)
import socket
original_getaddrinfo = socket.getaddrinfo

def ipv6_preferred_getaddrinfo(*args, **kwargs):
    results = original_getaddrinfo(*args, **kwargs)
    # Sort IPv6 first
    ipv6 = [r for r in results if r[0] == socket.AF_INET6]
    ipv4 = [r for r in results if r[0] == socket.AF_INET]
    return ipv6 + ipv4

socket.getaddrinfo = ipv6_preferred_getaddrinfo

# Create Service Bus client
client = ServiceBusClient.from_connection_string(connection_str)

with client:
    sender = client.get_queue_sender(queue_name="myqueue")
    with sender:
        message = ServiceBusMessage("Hello over IPv6!")
        sender.send_messages(message)
        print("Message sent over IPv6 Private Endpoint")
```

## Conclusion

Azure Service Bus IPv6 connectivity is achieved through Private Endpoints deployed in dual-stack subnets. The Private Endpoint gets both IPv4 and IPv6 addresses from the subnet, allowing clients in the VNet to connect over either protocol. Use private DNS zones to resolve the Service Bus namespace to the Private Endpoint addresses. For public access, configure IP filtering rules with IPv6 CIDR blocks. The Premium SKU is required for Private Endpoint support.
