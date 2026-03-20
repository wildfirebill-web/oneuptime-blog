# How to Set Up Azure CDN with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, CDN, Content Delivery, Infrastructure as Code, Azure Front Door, Performance

Description: Learn how to configure Azure CDN profiles and endpoints using OpenTofu for global content acceleration with custom domains, caching rules, and HTTPS enforcement.

---

Azure CDN accelerates static content delivery through a global network of points of presence (POPs). With OpenTofu, you define your CDN profile, endpoints, and routing rules as code, making content delivery configuration as manageable as your application infrastructure.

## Creating an Azure CDN Profile and Endpoint

```hcl
# main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "cdn" {
  name     = "cdn-${var.environment}-rg"
  location = var.location
}

# CDN Profile — defines the pricing tier (Standard Microsoft, Verizon, Akamai)
resource "azurerm_cdn_profile" "main" {
  name                = "${var.project_name}-cdn-profile"
  location            = "Global"  # CDN profiles are global resources
  resource_group_name = azurerm_resource_group.cdn.name
  sku                 = "Standard_Microsoft"  # Standard_Microsoft, Standard_Verizon, Premium_Verizon

  tags = var.common_tags
}

# CDN Endpoint pointing to a storage account origin
resource "azurerm_cdn_endpoint" "static_site" {
  name                = "${var.project_name}-static"
  profile_name        = azurerm_cdn_profile.main.name
  location            = "Global"
  resource_group_name = azurerm_resource_group.cdn.name

  # Enable HTTPS-only access
  is_http_allowed  = false
  is_https_allowed = true

  # Origin configuration
  origin {
    name       = "storage-origin"
    host_name  = azurerm_storage_account.static_site.primary_blob_host
    http_port  = 80
    https_port = 443
  }

  origin_host_header = azurerm_storage_account.static_site.primary_blob_host

  # Caching rules
  delivery_rule {
    name  = "CacheStaticFiles"
    order = 1

    request_uri_condition {
      operator = "EndsWith"
      match_values = [".js", ".css", ".png", ".jpg", ".gif", ".ico", ".woff2"]
    }

    cache_expiration_action {
      behavior = "Override"
      duration = "7.00:00:00"  # Cache for 7 days
    }
  }

  # Redirect HTTP to HTTPS
  delivery_rule {
    name  = "HTTPSRedirect"
    order = 2

    request_scheme_condition {
      operator     = "Equal"
      match_values = ["HTTP"]
    }

    url_redirect_action {
      redirect_type = "PermanentRedirect"
      protocol      = "Https"
    }
  }

  # Gzip compression for text content
  content_types_to_compress = [
    "application/javascript",
    "application/json",
    "text/css",
    "text/html",
    "text/plain",
  ]
  is_compression_enabled = true
}
```

## Adding a Custom Domain

```hcl
# custom_domain.tf
# Add a custom domain to the CDN endpoint
resource "azurerm_cdn_endpoint_custom_domain" "main" {
  name            = "custom-domain"
  cdn_endpoint_id = azurerm_cdn_endpoint.static_site.id
  host_name       = var.custom_domain

  # Enable HTTPS with CDN-managed certificate
  cdn_managed_https {
    certificate_type = "Dedicated"
    protocol_type    = "ServerNameIndication"
    tls_version      = "TLS12"
  }
}
```

## Azure Front Door (Modern Alternative)

For advanced routing, WAF, and global load balancing, use Azure Front Door Standard/Premium.

```hcl
# front_door.tf
resource "azurerm_cdn_frontdoor_profile" "main" {
  name                = "${var.project_name}-afd"
  resource_group_name = azurerm_resource_group.cdn.name
  sku_name            = "Standard_AzureFrontDoor"

  tags = var.common_tags
}

resource "azurerm_cdn_frontdoor_endpoint" "main" {
  name                     = "${var.project_name}-endpoint"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.main.id
}

resource "azurerm_cdn_frontdoor_origin_group" "main" {
  name                     = "origin-group"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.main.id

  load_balancing {}
}

resource "azurerm_cdn_frontdoor_origin" "app" {
  name                          = "app-origin"
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.main.id
  enabled                       = true
  host_name                     = var.origin_hostname
  origin_host_header            = var.origin_hostname
  priority                      = 1
  weight                        = 1000
  certificate_name_check_enabled = true
}
```

## Best Practices

- Use Azure Front Door Standard/Premium for new projects — it has more features than Classic CDN and includes WAF capabilities.
- Enable compression for text-based content types to reduce bandwidth and improve load times.
- Set long cache durations (7+ days) for immutable static assets with content-hashed filenames.
- Use HTTPS-only endpoints — HTTP serves no purpose for CDN-delivered content and leaks data in transit.
- Monitor CDN cache hit ratio — a ratio below 80% suggests caching rules need adjustment.
