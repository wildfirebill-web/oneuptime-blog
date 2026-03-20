# How to Create GCP Cloud DNS Record Sets with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Cloud DNS, DNS, Infrastructure as Code

Description: Learn how to create GCP Cloud DNS managed zones and record sets with OpenTofu for reliable, scalable DNS management on Google Cloud.

GCP Cloud DNS is a managed, authoritative DNS service. Managing DNS record sets in OpenTofu alongside your GCP infrastructure ensures DNS stays in sync as resources are created and updated.

## Provider Configuration

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = "us-central1"
}
```

## DNS Managed Zone

```hcl
resource "google_dns_managed_zone" "main" {
  name        = "example-com"
  dns_name    = "example.com."  # Trailing dot required
  description = "Primary DNS zone for example.com"

  dnssec_config {
    state = "on"  # Enable DNSSEC
  }
}

# Output nameservers to configure at your registrar
output "nameservers" {
  value = google_dns_managed_zone.main.name_servers
}
```

## A Record

```hcl
resource "google_dns_record_set" "www" {
  name         = "www.example.com."  # Trailing dot required
  managed_zone = google_dns_managed_zone.main.name
  type         = "A"
  ttl          = 300

  rrdatas = ["203.0.113.10"]
}
```

## A Record from GCP Resource

```hcl
resource "google_compute_global_address" "lb" {
  name = "lb-ip"
}

resource "google_dns_record_set" "apex" {
  name         = "example.com."
  managed_zone = google_dns_managed_zone.main.name
  type         = "A"
  ttl          = 60

  rrdatas = [google_compute_global_address.lb.address]
}
```

## CNAME Record

```hcl
resource "google_dns_record_set" "api" {
  name         = "api.example.com."
  managed_zone = google_dns_managed_zone.main.name
  type         = "CNAME"
  ttl          = 300

  rrdatas = ["api.example-origin.com."]  # Trailing dot on target too
}
```

## MX Records

```hcl
resource "google_dns_record_set" "mx" {
  name         = "example.com."
  managed_zone = google_dns_managed_zone.main.name
  type         = "MX"
  ttl          = 3600

  rrdatas = [
    "1 aspmx.l.google.com.",
    "5 alt1.aspmx.l.google.com.",
    "5 alt2.aspmx.l.google.com.",
    "10 alt3.aspmx.l.google.com.",
    "10 alt4.aspmx.l.google.com.",
  ]
}
```

## TXT Records

```hcl
resource "google_dns_record_set" "spf" {
  name         = "example.com."
  managed_zone = google_dns_managed_zone.main.name
  type         = "TXT"
  ttl          = 3600

  rrdatas = [
    "\"v=spf1 include:_spf.google.com ~all\"",  # Quotes required for TXT
    "\"google-site-verification=abc123\"",
  ]
}
```

## Private DNS Zone

```hcl
resource "google_dns_managed_zone" "private" {
  name        = "internal-example-com"
  dns_name    = "internal.example.com."
  description = "Private DNS zone for internal services"
  visibility  = "private"

  private_visibility_config {
    networks {
      network_url = google_compute_network.main.id
    }
  }
}

resource "google_dns_record_set" "db_internal" {
  name         = "db.internal.example.com."
  managed_zone = google_dns_managed_zone.private.name
  type         = "A"
  ttl          = 60
  rrdatas      = ["10.0.1.10"]
}
```

## Multiple Records with for_each

```hcl
locals {
  a_records = {
    "app.example.com."   = "203.0.113.10"
    "api.example.com."   = "203.0.113.20"
    "admin.example.com." = "203.0.113.30"
  }
}

resource "google_dns_record_set" "records" {
  for_each = local.a_records

  name         = each.key
  managed_zone = google_dns_managed_zone.main.name
  type         = "A"
  ttl          = 300
  rrdatas      = [each.value]
}
```

## Conclusion

GCP Cloud DNS record sets in OpenTofu keep your DNS configuration synchronized with your Google Cloud infrastructure. Always include trailing dots on FQDNs, reference GCP resource attributes directly rather than hardcoding IPs, and use private zones with network visibility for internal service discovery. Enable DNSSEC on your managed zones for added security.
