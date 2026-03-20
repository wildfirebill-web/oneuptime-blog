# How to Configure GCP Cloud DNS with AAAA Records

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, Cloud DNS, AAAA Records, DNS, Google Cloud

Description: Create and manage AAAA records in Google Cloud DNS for IPv6 address resolution, configure PTR records for reverse DNS, and set up private DNS zones with IPv6 support.

## Introduction

Google Cloud DNS supports IPv6 through AAAA records for forward resolution and PTR records in reverse zones for reverse DNS lookups. Cloud DNS managed zones work identically for IPv4 and IPv6 — you create AAAA records pointing hostnames to IPv6 addresses. Private zones support AAAA records for internal service discovery over IPv6.

## Create AAAA Records with gcloud

```bash
PROJECT="my-project"
ZONE_NAME="example-com"
DNS_NAME="example.com."

# Create a managed public DNS zone (if not existing)
gcloud dns managed-zones create "$ZONE_NAME" \
    --project="$PROJECT" \
    --dns-name="$DNS_NAME" \
    --description="Public zone for example.com"

# Add AAAA record for apex domain
gcloud dns record-sets create example.com. \
    --zone="$ZONE_NAME" \
    --type=AAAA \
    --ttl=300 \
    --rrdatas="2600:1900:4000:abc1:8000::" \
    --project="$PROJECT"

# Add AAAA record for www subdomain
gcloud dns record-sets create www.example.com. \
    --zone="$ZONE_NAME" \
    --type=AAAA \
    --ttl=300 \
    --rrdatas="2600:1900:4000:abc1:8001::" \
    --project="$PROJECT"

# Add both A and AAAA for dual-stack
gcloud dns record-sets create api.example.com. \
    --zone="$ZONE_NAME" \
    --type=A \
    --ttl=300 \
    --rrdatas="34.100.200.1" \
    --project="$PROJECT"

gcloud dns record-sets create api.example.com. \
    --zone="$ZONE_NAME" \
    --type=AAAA \
    --ttl=300 \
    --rrdatas="2600:1900:4000:abc1:8002::" \
    --project="$PROJECT"

# Verify records
gcloud dns record-sets list \
    --zone="$ZONE_NAME" \
    --project="$PROJECT" \
    --filter="type:AAAA"
```

## Terraform Cloud DNS with AAAA Records

```hcl
# cloud_dns_ipv6.tf

variable "project_id" {}

# Public managed zone
resource "google_dns_managed_zone" "public" {
  name        = "example-com"
  dns_name    = "example.com."
  project     = var.project_id
  description = "Public DNS zone"

  visibility = "public"
}

# A record (IPv4)
resource "google_dns_record_set" "a_apex" {
  name         = "example.com."
  managed_zone = google_dns_managed_zone.public.name
  project      = var.project_id
  type         = "A"
  ttl          = 300

  rrdatas = ["34.100.200.1"]
}

# AAAA record (IPv6) for apex
resource "google_dns_record_set" "aaaa_apex" {
  name         = "example.com."
  managed_zone = google_dns_managed_zone.public.name
  project      = var.project_id
  type         = "AAAA"
  ttl          = 300

  rrdatas = ["2600:1900:4000:abc1:8000::"]
}

# AAAA record for www
resource "google_dns_record_set" "aaaa_www" {
  name         = "www.example.com."
  managed_zone = google_dns_managed_zone.public.name
  project      = var.project_id
  type         = "AAAA"
  ttl          = 300

  rrdatas = ["2600:1900:4000:abc1:8001::"]
}

# Multiple AAAA records (round-robin)
resource "google_dns_record_set" "aaaa_api" {
  name         = "api.example.com."
  managed_zone = google_dns_managed_zone.public.name
  project      = var.project_id
  type         = "AAAA"
  ttl          = 60

  rrdatas = [
    "2600:1900:4000:abc1:8010::",
    "2600:1900:4000:abc1:8011::",
    "2600:1900:4000:abc1:8012::"
  ]
}
```

## Private DNS Zone with IPv6

```hcl
# Private zone for internal services
resource "google_dns_managed_zone" "private" {
  name        = "internal-example-com"
  dns_name    = "internal.example.com."
  project     = var.project_id
  description = "Private DNS zone for internal services"

  visibility = "private"

  private_visibility_config {
    networks {
      network_url = google_compute_network.main.id
    }
  }
}

# Internal AAAA record for service discovery
resource "google_dns_record_set" "aaaa_internal_service" {
  name         = "api.internal.example.com."
  managed_zone = google_dns_managed_zone.private.name
  project      = var.project_id
  type         = "AAAA"
  ttl          = 60

  # ULA IPv6 address for internal service
  rrdatas = ["fd20:0:0:1::10"]
}
```

## PTR Records for Reverse DNS

```bash
# Create reverse zone for IPv6 prefix 2600:1900:4000::/48
# Reverse zone name: 0.0.0.4.0.0.9.1.0.0.6.2.ip6.arpa.
gcloud dns managed-zones create ipv6-reverse \
    --project="$PROJECT" \
    --dns-name="0.0.0.4.0.0.9.1.0.0.6.2.ip6.arpa." \
    --description="IPv6 reverse zone"

# Add PTR record
# For address 2600:1900:4000:abc1:8000::
# Reversed: 0.0.0.0.0.0.0.8.1.c.b.a.0.0.0.4.0.0.9.1.0.0.6.2.ip6.arpa.
gcloud dns record-sets create \
    "0.0.0.0.0.0.0.8.1.c.b.a.0.0.0.4.0.0.9.1.0.0.6.2.ip6.arpa." \
    --zone=ipv6-reverse \
    --type=PTR \
    --ttl=300 \
    --rrdatas="www.example.com." \
    --project="$PROJECT"

# Verify reverse lookup
dig -x 2600:1900:4000:abc1:8000::
```

## Conclusion

GCP Cloud DNS AAAA records work exactly like A records — use `type = "AAAA"` in Terraform or `--type=AAAA` in gcloud with the IPv6 address as the record data. Create both A and AAAA records pointing to your IPv4 and IPv6 addresses for dual-stack name resolution. Private zones support AAAA records for internal service discovery using ULA addresses. For reverse DNS, create a PTR record in the appropriate `ip6.arpa.` reverse zone. Use `dig AAAA hostname` or `dig -x ipv6-address` to verify DNS resolution.
