# How to Set Up Azure Application Gateway for IPv4 Load Balancing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, Application Gateway, Load Balancing, IPv4, WAF, Networking

Description: Deploy and configure Azure Application Gateway to load balance HTTP/HTTPS traffic across IPv4 backend pools with health probes and routing rules.

## Introduction

Azure Application Gateway is a layer-7 load balancer with built-in SSL termination, URL-based routing, and optional Web Application Firewall (WAF). It distributes HTTP/HTTPS traffic to backend pools of VMs, scale sets, or IP addresses based on configurable routing rules.

## Prerequisites

- An Azure VNet with a dedicated subnet for Application Gateway (minimum /26 CIDR)
- Backend VMs or IPs to load balance
- Azure CLI or Terraform

## Creating the Application Gateway

```bash
# Step 1: Create a dedicated subnet for the gateway (must not contain other resources)

az network vnet subnet create \
  --resource-group rg-web \
  --vnet-name my-vnet \
  --name app-gw-subnet \
  --address-prefix 10.0.3.0/26

# Step 2: Create a public IP for the frontend
az network public-ip create \
  --resource-group rg-web \
  --name app-gw-pip \
  --sku Standard \
  --allocation-method Static

# Step 3: Create the Application Gateway
az network application-gateway create \
  --name my-app-gw \
  --resource-group rg-web \
  --location eastus \
  --vnet-name my-vnet \
  --subnet app-gw-subnet \
  --public-ip-address app-gw-pip \
  --sku Standard_v2 \
  --capacity 2 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --frontend-port 80 \
  --routing-rule-type Basic
```

## Adding Backend Pool Members

```bash
# Add backend servers (VMs by IP) to the backend pool
az network application-gateway address-pool update \
  --resource-group rg-web \
  --gateway-name my-app-gw \
  --name appGatewayBackendPool \
  --add backendAddresses ipAddress=10.0.1.10 \
  --add backendAddresses ipAddress=10.0.1.11
```

## Configuring Custom Health Probes

```bash
# Add a custom health probe for the backend
az network application-gateway probe create \
  --resource-group rg-web \
  --gateway-name my-app-gw \
  --name health-probe \
  --protocol Http \
  --host-name-from-http-settings true \
  --path /health \
  --interval 30 \
  --timeout 30 \
  --threshold 3

# Associate the probe with HTTP settings
az network application-gateway http-settings update \
  --resource-group rg-web \
  --gateway-name my-app-gw \
  --name appGatewayBackendHttpSettings \
  --probe health-probe
```

## URL-Based Routing

Route traffic to different backends based on the URL path:

```bash
# Create URL path map for path-based routing
az network application-gateway url-path-map create \
  --resource-group rg-web \
  --gateway-name my-app-gw \
  --name url-path-map \
  --paths /api/* \
  --rule-name api-rule \
  --address-pool api-backend-pool \
  --http-settings api-http-settings \
  --default-address-pool web-backend-pool \
  --default-http-settings web-http-settings
```

## Enabling HTTPS with SSL Termination

```bash
# Upload SSL certificate
az network application-gateway ssl-cert create \
  --resource-group rg-web \
  --gateway-name my-app-gw \
  --name my-ssl-cert \
  --cert-file cert.pfx \
  --cert-password MyPassword123

# Add HTTPS frontend listener
az network application-gateway frontend-port create \
  --resource-group rg-web \
  --gateway-name my-app-gw \
  --name port-443 \
  --port 443
```

## Terraform Configuration

```hcl
resource "azurerm_application_gateway" "main" {
  name                = "my-app-gw"
  resource_group_name = azurerm_resource_group.web.name
  location            = azurerm_resource_group.web.location

  sku {
    name     = "Standard_v2"
    tier     = "Standard_v2"
    capacity = 2
  }

  # ... (frontend_ip, backend_pool, http_settings, listener, routing_rule)
}
```

## Conclusion

Azure Application Gateway provides a fully managed layer-7 load balancer with health probes, SSL termination, and path-based routing. It is the right choice for HTTP/HTTPS workloads that need WAF protection or advanced routing capabilities.
