# How to Configure IPv6 Subnets in Google Cloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, Subnets, Dual-Stack, Google Cloud, VPC

Description: Create and configure Google Cloud VPC subnets with IPv6 support, choose between external and internal IPv6 access types, and understand GCP's /96 subnet allocation.

## Introduction

Google Cloud assigns IPv6 ranges to subnets when the `stack-type` is set to `IPV4_IPV6`. Each dual-stack subnet gets a `/96` IPv6 CIDR block from GCP's infrastructure. The `ipv6-access-type` determines whether addresses are globally routable (`EXTERNAL`) or only within the VPC (`INTERNAL`). Subnets can be converted from IPv4-only to dual-stack without recreating them.

## Create IPv6-Enabled Subnets

```bash
PROJECT="my-gcp-project"

# External dual-stack subnet (public IPv6)

gcloud compute networks subnets create subnet-public \
    --network=vpc-main \
    --region=us-east1 \
    --range=10.0.1.0/24 \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=EXTERNAL \
    --project="$PROJECT"

# Internal dual-stack subnet (private IPv6)
gcloud compute networks subnets create subnet-internal \
    --network=vpc-main \
    --region=us-east1 \
    --range=10.0.2.0/24 \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=INTERNAL \
    --project="$PROJECT"

# IPv6-only subnet
gcloud compute networks subnets create subnet-ipv6only \
    --network=vpc-main \
    --region=us-east1 \
    --stack-type=IPV6_ONLY \
    --ipv6-access-type=INTERNAL \
    --project="$PROJECT"

# View assigned IPv6 CIDRs
gcloud compute networks subnets describe subnet-public \
    --region=us-east1 \
    --project="$PROJECT" \
    --format="table(name, ipCidrRange, ipv6CidrRange, ipv6AccessType)"
```

## Update Existing Subnet to Add IPv6

```bash
# Enable IPv6 on existing IPv4-only subnet
gcloud compute networks subnets update existing-subnet \
    --region=us-east1 \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=EXTERNAL \
    --project="$PROJECT"

# Verify the update
gcloud compute networks subnets describe existing-subnet \
    --region=us-east1 \
    --project="$PROJECT" \
    --format="json(stackType, ipv6CidrRange, ipv6AccessType)"
```

## Terraform Subnets with IPv6 Options

```hcl
# subnets_ipv6.tf

variable "project_id" {}

# Public web subnet with external IPv6
resource "google_compute_subnetwork" "web" {
  name          = "subnet-web"
  ip_cidr_range = "10.0.1.0/24"
  region        = "us-east1"
  network       = google_compute_network.main.id
  project       = var.project_id

  # Dual-stack with globally routable IPv6
  stack_type       = "IPV4_IPV6"
  ipv6_access_type = "EXTERNAL"

  # Enable Private Google Access for IPv6
  private_ipv6_google_access = "ENABLE_OUTBOUND_VM_ACCESS_TO_GOOGLE"

  log_config {
    aggregation_interval = "INTERVAL_5_MIN"
    flow_sampling        = 1.0
    metadata             = "INCLUDE_ALL_METADATA"
  }
}

# Backend app subnet with internal IPv6
resource "google_compute_subnetwork" "app" {
  name          = "subnet-app"
  ip_cidr_range = "10.0.2.0/24"
  region        = "us-east1"
  network       = google_compute_network.main.id
  project       = var.project_id

  stack_type       = "IPV4_IPV6"
  ipv6_access_type = "INTERNAL"

  # Private Google Access for IPv6 (reach Google APIs over IPv6)
  private_ipv6_google_access = "ENABLE_OUTBOUND_VM_ACCESS_TO_GOOGLE"
}

# Database subnet with internal IPv6 only
resource "google_compute_subnetwork" "db" {
  name          = "subnet-db"
  ip_cidr_range = "10.0.3.0/24"
  region        = "us-east1"
  network       = google_compute_network.main.id
  project       = var.project_id

  # Internal only for security
  stack_type       = "IPV4_IPV6"
  ipv6_access_type = "INTERNAL"
}
```

## Understanding GCP IPv6 Subnet Allocation

```bash
# GCP subnet IPv6 allocation:
# When you create an external IPv6 subnet, GCP assigns a /48 prefix
# Each VM in the subnet gets a /96 from that /48
# Example:
#   Subnet gets: 2600:1900:4000:abc1::/48
#   VM 1 gets:   2600:1900:4000:abc1:8000::/96

# For internal IPv6:
# GCP assigns ULA /48 from fd00::/8 range
# Example:
#   Subnet gets: fd20:0000:0000::/48
#   VM 1 gets:   fd20:0000:0000:0001::/96

# View the allocated prefix for your subnet
gcloud compute networks subnets describe subnet-web \
    --region=us-east1 \
    --format="get(externalIpv6Prefix, ipv6CidrRange)"
```

## Private Google Access for IPv6

```bash
# Enable Private IPv6 Google Access on subnet
# This allows VMs to reach Google APIs over IPv6 without external IP
gcloud compute networks subnets update subnet-internal \
    --region=us-east1 \
    --private-ipv6-google-access=ENABLE_OUTBOUND_VM_ACCESS_TO_GOOGLE \
    --project="$PROJECT"

# Verify
gcloud compute networks subnets describe subnet-internal \
    --region=us-east1 \
    --format="get(privateIpv6GoogleAccess)"
```

## Conclusion

GCP dual-stack subnets require two settings: `stack-type=IPV4_IPV6` and `ipv6-access-type` (EXTERNAL or INTERNAL). External subnets receive GCP-managed globally routable `/48` prefixes, with VMs getting `/96` allocations. Internal subnets use ULA ranges for private VPC-only communication. Enable `private_ipv6_google_access` to allow VMs to reach Google APIs over IPv6 without public addresses. Existing subnets can be upgraded to dual-stack without recreation using `gcloud compute networks subnets update`.
