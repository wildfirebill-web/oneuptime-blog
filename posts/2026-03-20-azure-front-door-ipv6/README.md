# How to Enable IPv6 on Azure Front Door

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, IPv6, Front Door, CDN, Global Load Balancer, Anycast

Description: Configure Azure Front Door to accept IPv6 client connections, enabling global IPv6 anycast distribution of web applications and APIs.

## Introduction

Azure Front Door is a global HTTP/HTTPS load balancer and CDN that supports IPv6 by default. All Front Door endpoints accept IPv6 connections from clients because Front Door operates from Microsoft's anycast network which has global IPv6 presence. Backend origins can be IPv4-only - Front Door handles IPv6 termination and connects to backends over IPv4 or IPv6 as available.

## Azure Front Door Classic with IPv6

```bash
RG="rg-frontdoor"

# Front Door Classic (legacy) supports IPv6 by default

# Just create the profile and IPv6 works automatically

az network front-door create \
    --resource-group "$RG" \
    --name myapp-frontdoor \
    --backend-address mybackend.example.com \
    --frontend-host-name myapp.azurefd.net \
    --forwarding-protocol HttpsOnly

# Verify IPv6 access
dig AAAA myapp.azurefd.net
curl -6 https://myapp.azurefd.net/
```

## Azure Front Door Standard/Premium with IPv6

```hcl
# frontdoor_ipv6.tf (Standard/Premium tier)

resource "azurerm_cdn_frontdoor_profile" "main" {
  name                = "afd-main"
  resource_group_name = azurerm_resource_group.main.name
  sku_name            = "Standard_AzureFrontDoor"

  tags = { environment = "production" }
}

resource "azurerm_cdn_frontdoor_endpoint" "main" {
  name                     = "endpoint-web"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.main.id

  tags = { purpose = "web" }
}

resource "azurerm_cdn_frontdoor_origin_group" "web" {
  name                     = "origin-group-web"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.main.id

  load_balancing {
    sample_size                        = 4
    successful_samples_required        = 3
    additional_latency_in_milliseconds = 50
  }

  health_probe {
    path                = "/health"
    request_type        = "GET"
    protocol            = "Https"
    interval_in_seconds = 30
  }
}

resource "azurerm_cdn_frontdoor_origin" "web" {
  name                          = "origin-web-backend"
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.web.id
  enabled                       = true

  certificate_name_check_enabled = true
  host_name                      = azurerm_linux_virtual_machine.web.public_ip_address
  http_port                      = 80
  https_port                     = 443
  origin_host_header             = "www.example.com"
  priority                       = 1
  weight                         = 1000
}

resource "azurerm_cdn_frontdoor_route" "web" {
  name                          = "route-web"
  cdn_frontdoor_endpoint_id     = azurerm_cdn_frontdoor_endpoint.main.id
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.web.id
  cdn_frontdoor_origin_ids      = [azurerm_cdn_frontdoor_origin.web.id]

  supported_protocols    = ["Http", "Https"]
  patterns_to_match      = ["/*"]
  forwarding_protocol    = "HttpsOnly"
  https_redirect_enabled = true

  cache {
    query_string_caching_behavior = "IgnoreQueryString"
  }
}

output "frontdoor_endpoint" {
  value = azurerm_cdn_frontdoor_endpoint.main.host_name
}
```

## Verify IPv6 Front Door Access

```bash
# Check if Front Door has AAAA records
AFD_HOSTNAME="endpoint-web-xyz.z01.azurefd.net"

dig AAAA "$AFD_HOSTNAME"
# Expected: AAAA record with anycast IPv6 addresses

# Test IPv6 access
curl -6 -I "https://${AFD_HOSTNAME}/"
curl -6 -H "Host: www.example.com" "https://${AFD_HOSTNAME}/"

# Check which IP was used
curl -6 -w "Connected to: %{remote_ip}\n" -s -o /dev/null "https://${AFD_HOSTNAME}/"

# Test with custom domain
curl -6 -I https://www.example.com
```

## Custom Domain with IPv6 on Front Door

```bash
# Add custom domain to Front Door
az afd custom-domain create \
    --resource-group "$RG" \
    --profile-name afd-main \
    --custom-domain-name www-example \
    --host-name www.example.com \
    --minimum-tls-version TLS12 \
    --certificate-type ManagedCertificate

# Create DNS CNAME for custom domain → Front Door
# In Azure DNS:
az network dns record-set cname create \
    --resource-group "$RG" \
    --zone-name example.com \
    --name www \
    --ttl 3600

az network dns record-set cname set-record \
    --resource-group "$RG" \
    --zone-name example.com \
    --record-set-name www \
    --cname "endpoint-web-xyz.z01.azurefd.net"

# For IPv6 with CNAME:
# CNAME → Front Door hostname (has AAAA records)
# Clients follow CNAME → get IPv6 address from Front Door
```

## Conclusion

Azure Front Door supports IPv6 natively - all Front Door endpoints automatically respond to IPv6 clients through Microsoft's anycast network. No special IPv6 configuration is required. The Front Door domain names have AAAA records resolving to anycast IPv6 addresses. Custom domains using CNAME to the Front Door endpoint inherit IPv6 accessibility. Backends can be IPv4-only since Front Door terminates IPv6 connections at the edge and connects to origins over IPv4 or IPv6 as needed.
