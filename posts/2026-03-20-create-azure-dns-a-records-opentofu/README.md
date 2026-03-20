# How to Create Azure DNS A Records with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, DNS, Infrastructure as Code, Networking

Description: Learn how to create Azure DNS A records and manage DNS zones with OpenTofu for consistent DNS configuration across Azure-hosted services.

Azure DNS hosts DNS zones and provides name resolution for Azure resources. Managing DNS records in OpenTofu ensures your DNS configuration is version-controlled, consistently deployed, and updated automatically when service IPs change.

## Provider Configuration

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}
```

## DNS Zone

```hcl
resource "azurerm_resource_group" "dns" {
  name     = "dns-rg"
  location = "eastus"
}

resource "azurerm_dns_zone" "main" {
  name                = "example.com"
  resource_group_name = azurerm_resource_group.dns.name
}

# Output the nameservers to configure at your registrar
output "nameservers" {
  value = azurerm_dns_zone.main.name_servers
}
```

## A Record

```hcl
resource "azurerm_dns_a_record" "www" {
  name                = "www"
  zone_name           = azurerm_dns_zone.main.name
  resource_group_name = azurerm_resource_group.dns.name
  ttl                 = 300
  records             = ["203.0.113.10"]
}
```

## A Record Pointing to Azure Resource

```hcl
# Get the public IP from an existing Azure load balancer
data "azurerm_public_ip" "lb" {
  name                = "lb-public-ip"
  resource_group_name = "app-rg"
}

resource "azurerm_dns_a_record" "api" {
  name                = "api"
  zone_name           = azurerm_dns_zone.main.name
  resource_group_name = azurerm_resource_group.dns.name
  ttl                 = 60
  records             = [data.azurerm_public_ip.lb.ip_address]
}
```

## Alias Record (Pointing to Azure Resource)

```hcl
# Azure-native alias record (uses target_resource_id instead of records)
resource "azurerm_dns_a_record" "apex" {
  name                = "@"  # Zone apex
  zone_name           = azurerm_dns_zone.main.name
  resource_group_name = azurerm_resource_group.dns.name
  ttl                 = 300

  target_resource_id = azurerm_public_ip.frontend.id
}
```

## Multiple A Records

```hcl
locals {
  a_records = {
    "app"    = ["203.0.113.10", "203.0.113.11"]  # Multiple IPs for round-robin
    "api"    = ["203.0.113.20"]
    "admin"  = ["203.0.113.30"]
    "vpn"    = ["203.0.113.40"]
  }
}

resource "azurerm_dns_a_record" "records" {
  for_each = local.a_records

  name                = each.key
  zone_name           = azurerm_dns_zone.main.name
  resource_group_name = azurerm_resource_group.dns.name
  ttl                 = 300
  records             = each.value
}
```

## Private DNS Zone

```hcl
resource "azurerm_private_dns_zone" "internal" {
  name                = "internal.example.com"
  resource_group_name = azurerm_resource_group.dns.name
}

resource "azurerm_private_dns_a_record" "db" {
  name                = "db"
  zone_name           = azurerm_private_dns_zone.internal.name
  resource_group_name = azurerm_resource_group.dns.name
  ttl                 = 60
  records             = ["10.0.1.10"]
}

# Link private zone to a VNet
resource "azurerm_private_dns_zone_virtual_network_link" "main" {
  name                  = "vnet-link"
  resource_group_name   = azurerm_resource_group.dns.name
  private_dns_zone_name = azurerm_private_dns_zone.internal.name
  virtual_network_id    = azurerm_virtual_network.main.id
  registration_enabled  = false  # Manual registration; set true for auto-registration
}
```

## AAAA Record (IPv6)

```hcl
resource "azurerm_dns_aaaa_record" "www_ipv6" {
  name                = "www"
  zone_name           = azurerm_dns_zone.main.name
  resource_group_name = azurerm_resource_group.dns.name
  ttl                 = 300
  records             = ["2001:db8::1"]
}
```

## Conclusion

Azure DNS records in OpenTofu keep your DNS configuration synchronized with your Azure infrastructure. Reference public IP resources directly rather than hardcoding IPs to ensure DNS automatically reflects changes. Use private DNS zones with VNet links for internal service discovery, and use for_each to manage multiple records without repetition.
