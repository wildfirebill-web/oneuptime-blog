# How to Configure GCP VPC Network ULA Internal IPv6 Ranges

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, ULA, Internal IPv6, RFC 4193, Google Cloud, VPC

Description: Understand and configure Unique Local Addresses (ULA) for internal IPv6 in Google Cloud VPC, including how GCP assigns fd::/8 prefixes and when to use ULA vs globally routable addresses.

## Introduction

Google Cloud's internal IPv6 subnets use Unique Local Addresses (ULA) from the `fd00::/8` range as defined by RFC 4193. When you create a subnet with `ipv6-access-type=INTERNAL`, GCP automatically assigns a unique `/48` prefix from the ULA space. These addresses are only routable within the VPC and connected networks — they cannot reach or be reached from the public internet. ULA is ideal for backend services, databases, and inter-service communication that should not be internet-accessible.

## Create Subnets with ULA IPv6

```bash
PROJECT="my-project"
REGION="us-east1"

# Create internal (ULA) IPv6 subnet
gcloud compute networks subnets create subnet-db \
    --project="$PROJECT" \
    --network=vpc-main \
    --region="$REGION" \
    --range=10.0.3.0/24 \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=INTERNAL

# View the ULA /48 prefix GCP assigned
gcloud compute networks subnets describe subnet-db \
    --region="$REGION" \
    --project="$PROJECT" \
    --format="get(ipv6CidrRange)"
# Returns: fd20:0:0:3::/64  (example ULA prefix)

# Create multiple internal subnets — each gets a unique ULA /48
gcloud compute networks subnets create subnet-app \
    --project="$PROJECT" \
    --network=vpc-main \
    --region=us-west1 \
    --range=10.1.1.0/24 \
    --stack-type=IPV4_IPV6 \
    --ipv6-access-type=INTERNAL

gcloud compute networks subnets describe subnet-app \
    --region=us-west1 \
    --project="$PROJECT" \
    --format="get(ipv6CidrRange)"
# Returns a different ULA prefix — GCP ensures uniqueness
```

## Understand ULA Prefix Assignment

```bash
# GCP ULA prefix assignment:
# - GCP assigns a unique /48 per VPC from the fd::/8 range
# - Each subnet in the VPC gets a /64 from that /48
# - VMs in the subnet get a /96 from the /64
#
# Example VPC ULA hierarchy:
#   VPC:    fd20:0000:0000::/48   (GCP-assigned)
#   Subnet1: fd20:0000:0000:0001::/64
#   Subnet2: fd20:0000:0000:0002::/64
#   VM1:    fd20:0000:0000:0001:8000::/96

# List all subnets and their ULA ranges
gcloud compute networks subnets list \
    --project="$PROJECT" \
    --filter="ipv6AccessType=INTERNAL" \
    --format="table(name, region, ipv6CidrRange, ipv6AccessType)"

# ULA address characteristics:
# - Starts with fd (binary: 1111 1101)
# - Globally unique prefix (40 random bits)
# - Not routable on public internet
# - Consistent routing within VPC and peered networks
```

## Routing with ULA Addresses

```bash
# ULA addresses route automatically within the VPC
# No additional routes needed for intra-VPC communication

# For VPC peering with ULA: import/export routes
gcloud compute networks peerings create peer-vpc-a-b \
    --project="$PROJECT" \
    --network=vpc-main \
    --peer-network=vpc-peer \
    --export-custom-routes \
    --import-custom-routes \
    --exchange-subnet-routes \
    --stack-type=IPV4_IPV6  # Include IPv6 routes in peering

# For outbound internet from ULA VMs: Cloud NAT is required
# (ULA addresses are not routable on the internet)

# Firewall rule to allow inter-VPC ULA traffic
gcloud compute firewall-rules create allow-ula-internal \
    --project="$PROJECT" \
    --network=vpc-main \
    --direction=INGRESS \
    --source-ranges="fd00::/8" \
    --rules=all
```

## Terraform ULA Subnets

```hcl
# ula_subnets.tf

variable "project_id" {}

resource "google_compute_network" "main" {
  name                    = "vpc-main"
  auto_create_subnetworks = false
  project                 = var.project_id
}

# Database subnet — internal ULA only (most secure)
resource "google_compute_subnetwork" "db" {
  name          = "subnet-db"
  ip_cidr_range = "10.0.3.0/24"
  region        = "us-east1"
  network       = google_compute_network.main.id
  project       = var.project_id

  stack_type       = "IPV4_IPV6"
  ipv6_access_type = "INTERNAL"  # ULA addresses
}

# App subnet — internal ULA
resource "google_compute_subnetwork" "app" {
  name          = "subnet-app"
  ip_cidr_range = "10.0.2.0/24"
  region        = "us-east1"
  network       = google_compute_network.main.id
  project       = var.project_id

  stack_type       = "IPV4_IPV6"
  ipv6_access_type = "INTERNAL"

  # Allow VMs to reach Google APIs over IPv6
  private_ipv6_google_access = "ENABLE_OUTBOUND_VM_ACCESS_TO_GOOGLE"
}

# Output the ULA ranges assigned by GCP
output "db_ula_range" {
  value = google_compute_subnetwork.db.ipv6_cidr_range
}

output "app_ula_range" {
  value = google_compute_subnetwork.app.ipv6_cidr_range
}
```

## Testing ULA Connectivity

```bash
# Test connectivity between VMs using ULA addresses
# SSH into VM in subnet-app
gcloud compute ssh vm-app --zone=us-east1-b --project="$PROJECT"

# Get ULA address of target VM in subnet-db
DB_IPV6=$(gcloud compute instances describe vm-db \
    --zone=us-east1-b \
    --project="$PROJECT" \
    --format="get(networkInterfaces[0].ipv6Address)")

# Test connectivity over ULA IPv6
ping6 -c 3 "$DB_IPV6"
curl -6 "http://[$DB_IPV6]:5432/"

# ULA addresses cannot reach the internet
ping6 -c 3 2001:4860:4860::8888  # This will fail without Cloud NAT
```

## Conclusion

GCP ULA IPv6 addresses (internal IPv6) are automatically assigned from the `fd00::/8` range when subnets use `ipv6-access-type=INTERNAL`. GCP guarantees uniqueness within your VPC by assigning a random 40-bit global ID. ULA addresses route freely within the VPC and peered networks but require Cloud NAT for internet access. Use ULA for backend services, databases, and microservices that should not be internet-accessible. Enable `private_ipv6_google_access` on ULA subnets to allow VMs to reach Google APIs over IPv6 without external addresses.
