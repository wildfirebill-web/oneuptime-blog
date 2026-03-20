# How to Create Azure CDN Endpoints with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, CDN, Content Delivery, Infrastructure as Code, Performance

Description: Learn how to create Azure CDN profiles and endpoints for static content delivery and dynamic site acceleration using OpenTofu.

## Introduction

Azure CDN caches static assets at Points of Presence (PoPs) worldwide, reducing latency for global users. OpenTofu manages CDN profiles, endpoints, and custom domain configurations as code.

## Creating a CDN Profile

```hcl
resource "azurerm_resource_group" "cdn" {
  name     = "rg-cdn-${var.environment}"
  location = var.location
}

resource "azurerm_cdn_profile" "main" {
  name                = "cdnp-${var.app_name}-${var.environment}"
  location            = azurerm_resource_group.cdn.location
  resource_group_name = azurerm_resource_group.cdn.name

  # SKU options: Standard_Akamai, Standard_ChinaCdn, Standard_Microsoft,
  # Standard_Verizon, Premium_Verizon
  sku = "Standard_Microsoft"

  tags = {
    Environment = var.environment
    ManagedBy   = "opentofu"
  }
}
```

## Creating a CDN Endpoint

```hcl
resource "azurerm_cdn_endpoint" "static" {
  name                = "cdne-${var.app_name}-${var.environment}"
  profile_name        = azurerm_cdn_profile.main.name
  location            = azurerm_resource_group.cdn.location
  resource_group_name = azurerm_resource_group.cdn.name

  # Origin – can be storage, web app, or custom hostname
  origin {
    name      = "storage-origin"
    host_name = azurerm_storage_account.static.primary_blob_host
    http_port  = 80
    https_port = 443
  }

  origin_host_header = azurerm_storage_account.static.primary_blob_host

  # Optimization for static web delivery
  optimization_type = "GeneralWebDelivery"

  is_http_allowed  = false  # HTTPS only
  is_https_allowed = true

  # Caching rules
  delivery_rule {
    name  = "cache-static-assets"
    order = 1

    query_string_condition {
      operator         = "Any"
      negate_condition = false
    }

    url_path_condition {
      operator         = "BeginsWith"
      match_values     = ["/static/"]
      negate_condition = false
    }

    cache_expiration_action {
      behavior = "Override"
      duration = "7.00:00:00"  # 7 days
    }
  }

  # HTTPS redirect rule
  delivery_rule {
    name  = "https-redirect"
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

  tags = {
    Environment = var.environment
    ManagedBy   = "opentofu"
  }
}
```

## Storage Account Origin

```hcl
resource "azurerm_storage_account" "static" {
  name                     = "st${var.app_name}${var.environment}"
  resource_group_name      = azurerm_resource_group.cdn.name
  location                 = azurerm_resource_group.cdn.location
  account_tier             = "Standard"
  account_replication_type = "LRS"

  static_website {
    index_document     = "index.html"
    error_404_document = "index.html"
  }
}
```

## Custom Domain

```hcl
resource "azurerm_cdn_endpoint_custom_domain" "main" {
  name            = "custom-domain"
  cdn_endpoint_id = azurerm_cdn_endpoint.static.id
  host_name       = "cdn.example.com"

  cdn_managed_https {
    certificate_type = "Dedicated"
    protocol_type    = "ServerNameIndication"
    tls_version      = "TLS12"
  }
}
```

## Outputs

```hcl
output "cdn_endpoint_hostname" {
  value = azurerm_cdn_endpoint.static.host_name
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

Azure CDN provides global content delivery with configurable caching rules and custom domain support. OpenTofu manages CDN profiles, endpoints, delivery rules, and custom domain HTTPS — creating a reproducible, code-driven CDN configuration for static and dynamic content.
