# How to Configure IPv6 for Azure Container Instances

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, IPv6, Container Instances, ACI, Container, VNet Integration

Description: Configure Azure Container Instances with IPv6 connectivity through VNet integration, enabling containers to communicate over IPv6 and receive IPv6 addresses.

## Introduction

Azure Container Instances (ACI) support IPv6 through VNet-integrated deployments. When you deploy ACI containers in a dual-stack subnet, they receive IPv6 addresses from the subnet's IPv6 CIDR block. Public ACI deployments (non-VNet) are IPv4-only, so IPv6 requires VNet integration. This enables IPv6 inter-container communication and access to IPv6 services.

## Deploy Container Instance with IPv6 in VNet

```bash
RG="rg-aci-ipv6"
LOCATION="eastus"

# Create resource group

az group create --name "$RG" --location "$LOCATION"

# Create VNet with IPv6 support
az network vnet create \
    --resource-group "$RG" \
    --name vnet-aci \
    --address-prefixes "10.0.0.0/16" "fd00:aci::/48"

# Create subnet for ACI (must have delegations)
az network vnet subnet create \
    --resource-group "$RG" \
    --vnet-name vnet-aci \
    --name subnet-aci \
    --address-prefixes "10.0.1.0/24" "fd00:aci:0:1::/64" \
    --delegations "Microsoft.ContainerInstance/containerGroups"

SUBNET_ID=$(az network vnet subnet show \
    --resource-group "$RG" \
    --vnet-name vnet-aci \
    --name subnet-aci \
    --query id --output tsv)

# Deploy container group in VNet subnet
az container create \
    --resource-group "$RG" \
    --name aci-web \
    --image nginx:latest \
    --cpu 1 \
    --memory 1.5 \
    --vnet vnet-aci \
    --subnet "$SUBNET_ID" \
    --ports 80

# Get container IPv4 address
az container show \
    --resource-group "$RG" \
    --name aci-web \
    --query "ipAddress"
```

## Terraform ACI with IPv6

```hcl
# aci_ipv6.tf

resource "azurerm_container_group" "web" {
  name                = "aci-web"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  ip_address_type     = "Private"  # Required for VNet integration
  os_type             = "Linux"

  # VNet integration gives IPv6 from the subnet
  subnet_ids = [azurerm_subnet.aci.id]

  container {
    name   = "nginx"
    image  = "nginx:latest"
    cpu    = "0.5"
    memory = "1.5"

    ports {
      port     = 80
      protocol = "TCP"
    }

    # Environment for IPv6 configuration
    environment_variables = {
      ENABLE_IPV6 = "true"
    }
  }

  tags = { Name = "aci-web" }
}

# Subnet with ACI delegation and IPv6
resource "azurerm_subnet" "aci" {
  name                 = "subnet-aci"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name

  address_prefixes = [
    "10.0.1.0/24",
    "fd00:aci:0:1::/64",
  ]

  delegation {
    name = "aci-delegation"
    service_delegation {
      name    = "Microsoft.ContainerInstance/containerGroups"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/action",
      ]
    }
  }
}
```

## Verify IPv6 Inside ACI Container

```bash
# Execute into container to check IPv6
az container exec \
    --resource-group "$RG" \
    --name aci-web \
    --container-name nginx \
    --exec-command "/bin/sh"

# Inside container:
ip -6 addr show
# Should show fd00:aci:0:1:: IPv6 address

# Test IPv6 connectivity
curl -6 https://ipv6.google.com
ping6 -c 3 2001:4860:4860::8888

# Check DNS resolves AAAA records
nslookup -type=AAAA google.com
```

## Multi-Container Group with IPv6

```yaml
# aci-group.yaml - Multi-container group with IPv6
apiVersion: '2021-10-01'
location: eastus
name: aci-multi-container
properties:
  containers:
  - name: web
    properties:
      image: nginx:latest
      ports:
      - port: 80
      resources:
        requests:
          cpu: 0.5
          memoryInGb: 0.5
  - name: sidecar
    properties:
      image: myapp/sidecar:latest
      resources:
        requests:
          cpu: 0.25
          memoryInGb: 0.25
      environmentVariables:
      - name: USE_IPV6
        value: "true"
  ipAddress:
    type: Private
    ports:
    - port: 80
  subnetIds:
  - id: /subscriptions/xxx/resourceGroups/rg-aci-ipv6/providers/Microsoft.Network/virtualNetworks/vnet-aci/subnets/subnet-aci
  osType: Linux
type: Microsoft.ContainerInstance/containerGroups
```

## Conclusion

Azure Container Instances support IPv6 only through VNet-integrated deployments (not public/standalone ACI). Deploy containers in dual-stack subnets delegated to `Microsoft.ContainerInstance/containerGroups` to receive IPv6 addresses. Inside the container, the IPv6 address appears on the container's network interface. ACI containers in IPv6-enabled subnets can initiate IPv6 connections to other Azure services or the internet via the VNet's routing and Egress-Only IGW equivalent. Public-facing IPv6 for ACI requires a load balancer or Application Gateway in front of the VNet-integrated container group.
