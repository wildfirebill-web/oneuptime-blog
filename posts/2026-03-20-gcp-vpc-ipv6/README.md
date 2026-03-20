# How to Enable IPv6 in Google Cloud VPC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, VPC, Google Cloud, Dual-Stack, Cloud Networking

Description: Enable IPv6 in Google Cloud VPC networks, configure dual-stack subnets with internal and external IPv6 ranges, and understand GCP's unique IPv6 architecture.

## Introduction

Google Cloud Platform (GCP) supports IPv6 in VPC networks through subnet-level IPv6 ranges. Unlike AWS and Azure that assign IPv6 at the VPC level, GCP enables IPv6 per-subnet with the choice of external IPv6 (globally routable from the internet) or internal IPv6 (ULA, only within VPC). GCP VMs can receive dual-stack addresses with both IPv4 and IPv6.

## Enable IPv6 on GCP Subnet

```bash
PROJECT="my-project"
REGION="us-east1"
VPC_NAME="vpc-main"
SUBNET_NAME="subnet-web"

# Create VPC first (if not exists)
gcloud compute networks create "$VPC_NAME" \
    --subnet-mode=custom \
    --project="$PROJECT"

# Create subnet with external IPv6 (globally routable)
gcloud compute networks subnets create "$SUBNET_NAME" \
    --network="$VPC_NAME" \
    --region="$REGION" \
    --range="10.0.1.0/24" \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=EXTERNAL \
    --project="$PROJECT"

# Create subnet with internal IPv6 (ULA, private)
gcloud compute networks subnets create subnet-private \
    --network="$VPC_NAME" \
    --region="$REGION" \
    --range="10.0.2.0/24" \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=INTERNAL \
    --project="$PROJECT"

# View subnet IPv6 configuration
gcloud compute networks subnets describe "$SUBNET_NAME" \
    --region="$REGION" \
    --format="json(ipCidrRange, ipv6CidrRange, stackType, ipv6AccessType)" \
    --project="$PROJECT"
```

## Terraform GCP VPC with IPv6

```hcl
# gcp_vpc_ipv6.tf

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
  region  = "us-east1"
}

resource "google_compute_network" "main" {
  name                    = "vpc-main"
  auto_create_subnetworks = false
}

# External IPv6 subnet (globally routable)
resource "google_compute_subnetwork" "web" {
  name          = "subnet-web"
  ip_cidr_range = "10.0.1.0/24"
  region        = "us-east1"
  network       = google_compute_network.main.id

  # Enable dual-stack with external IPv6
  stack_type       = "IPV4_IPV6"
  ipv6_access_type = "EXTERNAL"

  log_config {
    aggregation_interval = "INTERVAL_10_MIN"
    flow_sampling        = 0.5
    metadata             = "INCLUDE_ALL_METADATA"
  }
}

# Internal IPv6 subnet (ULA only)
resource "google_compute_subnetwork" "app" {
  name          = "subnet-app"
  ip_cidr_range = "10.0.2.0/24"
  region        = "us-east1"
  network       = google_compute_network.main.id

  stack_type       = "IPV4_IPV6"
  ipv6_access_type = "INTERNAL"
}

output "web_subnet_ipv6" {
  value = google_compute_subnetwork.web.ipv6_cidr_range
}

output "app_subnet_ipv6" {
  value = google_compute_subnetwork.app.ipv6_cidr_range
}
```

## GCP IPv6 Architecture Concepts

```
GCP IPv6 Types:
  External IPv6 — GCP-assigned globally routable /96 prefix
    → VMs get addresses reachable from internet
    → Subnet gets /48 from GCP's public IPv6 range
    → Used for internet-facing workloads

  Internal IPv6 — ULA (Unique Local Addresses) /96
    → VMs get addresses only reachable within VPC
    → Good for backend services not exposed to internet
    → Unique per-subnet, not globally routable

Stack Types:
  IPV4_ONLY    — Only IPv4 (default)
  IPV4_IPV6    — Both IPv4 and IPv6 (dual-stack)
  IPV6_ONLY    — Only IPv6 (requires DNS64/NAT64)
```

## Verify VPC IPv6 Configuration

```bash
# List all subnets with IPv6 status
gcloud compute networks subnets list \
    --network="$VPC_NAME" \
    --format="table(name, region, ipCidrRange, ipv6CidrRange, stackType, ipv6AccessType)" \
    --project="$PROJECT"

# Check GCP-assigned IPv6 range for subnet
gcloud compute networks subnets describe "$SUBNET_NAME" \
    --region="$REGION" \
    --project="$PROJECT" \
    --format="get(ipv6CidrRange)"
# Returns something like: 2600:1900:4000:abcd::/64
```

## Conclusion

GCP VPC IPv6 is configured per-subnet using `stack_type = "IPV4_IPV6"` and `ipv6_access_type` of either `EXTERNAL` (globally routable) or `INTERNAL` (ULA). Unlike AWS where VPCs get a `/56`, GCP assigns individual `/96` or larger prefixes to subnets. External IPv6 subnets receive a `/48` prefix from GCP's public IP space, with individual VMs getting `/96` addresses. Internal IPv6 uses ULA ranges for private inter-VPC communication. After enabling IPv6 on a subnet, VMs in that subnet can be configured to receive IPv6 addresses via their network interface settings.
