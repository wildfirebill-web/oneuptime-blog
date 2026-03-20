# How to Set Up Azure Storage Static Website Hosting with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, Storage, Static Website, OpenTofu, CDN, Frontend

Description: Learn how to configure Azure Storage static website hosting with OpenTofu, including custom domain setup and Azure CDN integration for fast global delivery.

## Overview

Azure Storage supports hosting static websites (HTML, CSS, JavaScript) directly from a storage account. When combined with Azure CDN, you get a scalable, low-cost hosting solution for SPAs and static sites.

## Step 1: Create Storage Account with Static Website

```hcl
# main.tf - Storage account with static website hosting enabled
resource "azurerm_storage_account" "static_site" {
  name                     = "mystaticsitehosting"
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"

  # Enable static website hosting
  static_website {
    index_document     = "index.html"
    error_404_document = "404.html"
  }
}
```

## Step 2: Upload Static Files

```hcl
# Upload the index.html file to the $web container
resource "azurerm_storage_blob" "index" {
  name                   = "index.html"
  storage_account_name   = azurerm_storage_account.static_site.name
  storage_container_name = "$web"  # Special container for static websites
  type                   = "Block"
  source                 = "${path.module}/dist/index.html"
  content_type           = "text/html"
}

resource "azurerm_storage_blob" "not_found" {
  name                   = "404.html"
  storage_account_name   = azurerm_storage_account.static_site.name
  storage_container_name = "$web"
  type                   = "Block"
  source                 = "${path.module}/dist/404.html"
  content_type           = "text/html"
}
```

## Step 3: Set Up Azure CDN in Front of Storage

```hcl
# Create a CDN profile for global content delivery
resource "azurerm_cdn_profile" "cdn" {
  name                = "my-static-site-cdn"
  location            = "global"
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "Standard_Microsoft"
}

# CDN endpoint points to the storage account's static website endpoint
resource "azurerm_cdn_endpoint" "static_site_endpoint" {
  name                = "my-static-site-endpoint"
  profile_name        = azurerm_cdn_profile.cdn.name
  location            = "global"
  resource_group_name = azurerm_resource_group.rg.name

  # The origin is the storage static website URL (not the blob endpoint)
  origin {
    name      = "storage-origin"
    host_name = azurerm_storage_account.static_site.primary_web_host
  }

  origin_host_header = azurerm_storage_account.static_site.primary_web_host

  # Cache static assets aggressively
  delivery_rule {
    name  = "cache-assets"
    order = 1

    request_uri_condition {
      operator = "Contains"
      match_values = ["/assets/"]
    }

    cache_expiration_action {
      behavior = "Override"
      duration = "7.00:00:00"  # Cache for 7 days
    }
  }
}
```

## Step 4: Custom Domain with HTTPS

```hcl
# Add a custom domain to the CDN endpoint
resource "azurerm_cdn_endpoint_custom_domain" "custom_domain" {
  name            = "www-example-com"
  cdn_endpoint_id = azurerm_cdn_endpoint.static_site_endpoint.id
  host_name       = "www.example.com"

  # Enable Microsoft-managed HTTPS certificate
  cdn_managed_https {
    certificate_type = "Dedicated"
    protocol_type    = "ServerNameIndication"
    tls_version      = "TLS12"
  }
}
```

## Step 5: Outputs

```hcl
output "static_website_url" {
  value       = azurerm_storage_account.static_site.primary_web_endpoint
  description = "Direct storage static website URL"
}

output "cdn_endpoint_url" {
  value       = "https://${azurerm_cdn_endpoint.static_site_endpoint.host_name}"
  description = "CDN endpoint URL for the static website"
}
```

## Summary

Azure Storage static website hosting with CDN, managed through OpenTofu, provides a cost-effective and scalable solution for serving static web applications. The CDN layer adds global performance, custom domain support, and HTTPS without needing a web server.
