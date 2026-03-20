# How to Manage Cloudflare AAAA Records with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cloudflare, Terraform, IPv6, DNS, AAAA, Infrastructure as Code

Description: A guide to creating, updating, and managing Cloudflare AAAA DNS records for IPv6 hosts using the Terraform Cloudflare provider.

Cloudflare DNS supports AAAA records for IPv6 addresses. Managing these records with Terraform enables version-controlled, repeatable DNS configuration for your IPv6-enabled services.

## Step 1: Configure the Cloudflare Provider

```hcl
# provider.tf
terraform {
  required_providers {
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}

provider "cloudflare" {
  # Set CLOUDFLARE_API_TOKEN env var or use api_token attribute
  api_token = var.cloudflare_api_token
}

variable "cloudflare_api_token" {
  type      = string
  sensitive = true
}
```

## Step 2: Look Up the Zone ID

```hcl
# zone.tf - Look up the Cloudflare zone by domain name
data "cloudflare_zone" "main" {
  name = "example.com"
}

output "zone_id" {
  value = data.cloudflare_zone.main.id
}
```

## Step 3: Create a Simple AAAA Record

```hcl
# aaaa-record.tf - A single AAAA record for the root domain
resource "cloudflare_record" "root_aaaa" {
  zone_id = data.cloudflare_zone.main.id
  name    = "@"        # @ means the root domain (example.com)
  type    = "AAAA"
  value   = "2001:db8::1"  # The IPv6 address of your server
  ttl     = 300
  proxied = true     # Route through Cloudflare's IPv6 proxy/CDN
}
```

## Step 4: Create AAAA Records for Multiple Subdomains

```hcl
# multiple-aaaa.tf - AAAA records for several subdomains
locals {
  ipv6_records = {
    "www"    = "2001:db8::1"
    "api"    = "2001:db8::2"
    "mail"   = "2001:db8::3"
    "cdn"    = "2001:db8::4"
  }
}

resource "cloudflare_record" "aaaa_subdomains" {
  for_each = local.ipv6_records

  zone_id = data.cloudflare_zone.main.id
  name    = each.key         # subdomain name
  type    = "AAAA"
  value   = each.value       # IPv6 address
  ttl     = 1                # TTL of 1 = Auto when proxied
  proxied = true
}
```

## Step 5: Create Unproxied AAAA Records (DNS-Only)

For services that need direct IPv6 connectivity without Cloudflare's proxy:

```hcl
# dns-only-aaaa.tf - AAAA record bypassing Cloudflare proxy (DNS only)
resource "cloudflare_record" "mx_server_aaaa" {
  zone_id = data.cloudflare_zone.main.id
  name    = "mail"
  type    = "AAAA"
  value   = "2001:db8::10"

  # proxied = false means DNS-only (orange cloud off)
  proxied = false
  ttl     = 3600
}
```

## Step 6: Use Dynamic IPv6 from Infrastructure

Combine with AWS/GCP/Azure outputs to automatically set AAAA records:

```hcl
# dynamic-aaaa.tf - Use IPv6 from a GCP load balancer output as AAAA record
resource "cloudflare_record" "app_aaaa" {
  zone_id = data.cloudflare_zone.main.id
  name    = "app"
  type    = "AAAA"

  # Reference the IPv6 output from the GCP load balancer
  value   = google_compute_global_forwarding_rule.ipv6.ip_address

  proxied = false
  ttl     = 300
}
```

## Step 7: Apply and Verify

```bash
terraform apply

# Verify the AAAA record in Cloudflare
curl -X GET "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records?type=AAAA" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" | jq '.result[].name,.result[].content'

# Test DNS resolution
dig AAAA www.example.com @1.1.1.1
```

Managing Cloudflare AAAA records with Terraform ensures your IPv6 DNS configuration is consistent, auditable, and automatically updated whenever your underlying IPv6 infrastructure changes.
