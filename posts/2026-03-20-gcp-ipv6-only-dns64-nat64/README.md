# How to Configure IPv6-Only Subnets with DNS64/NAT64 on GCP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv6, DNS64, NAT64, IPv6-Only, Google Cloud, VPC

Description: Configure IPv6-only subnets on Google Cloud with DNS64 and NAT64 to allow IPv6-only VMs to communicate with IPv4-only services using protocol translation.

## Introduction

GCP supports IPv6-only subnets (`stack-type=IPV6_ONLY`) where VMs receive only IPv6 addresses. To allow these VMs to reach IPv4-only internet services, GCP provides DNS64 and NAT64: DNS64 synthesizes AAAA records for IPv4-only domains, mapping them to the NAT64 prefix `64:ff9b::/96`, and NAT64 translates outbound IPv6 packets to IPv4. This is the recommended path for new workloads that want to avoid dual-stack complexity while maintaining compatibility with IPv4 internet services.

## Create IPv6-Only Subnet

```bash
PROJECT="my-project"
REGION="us-east1"

# Create VPC with custom mode

gcloud compute networks create vpc-ipv6only \
    --project="$PROJECT" \
    --subnet-mode=custom

# Create IPv6-only subnet with internal access
gcloud compute networks subnets create subnet-ipv6only \
    --project="$PROJECT" \
    --network=vpc-ipv6only \
    --region="$REGION" \
    --stack-type=IPV6_ONLY \
    --ipv6-access-type=INTERNAL \
    --range=10.0.1.0/24  # IPv4 range still required for subnet definition

# View the IPv6-only subnet
gcloud compute networks subnets describe subnet-ipv6only \
    --region="$REGION" \
    --project="$PROJECT" \
    --format="json(stackType, ipv6CidrRange, ipv6AccessType)"

# Create VM in IPv6-only subnet
gcloud compute instances create vm-ipv6only \
    --project="$PROJECT" \
    --zone=us-east1-b \
    --machine-type=n2-standard-2 \
    --subnet=subnet-ipv6only \
    --stack-type=IPV6_ONLY \
    --ipv6-network-tier=PREMIUM \
    --image-family=debian-12 \
    --image-project=debian-cloud
```

## Configure Cloud NAT for NAT64

```bash
# Cloud Router is required for Cloud NAT
gcloud compute routers create router-ipv6only \
    --project="$PROJECT" \
    --network=vpc-ipv6only \
    --region="$REGION"

# Configure Cloud NAT with NAT64 support
# When the subnet is IPv6-only, Cloud NAT automatically provides NAT64
gcloud compute routers nats create nat-nat64 \
    --project="$PROJECT" \
    --router=router-ipv6only \
    --region="$REGION" \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips \
    --enable-endpoint-independent-mapping

# Verify NAT configuration
gcloud compute routers nats describe nat-nat64 \
    --router=router-ipv6only \
    --region="$REGION" \
    --project="$PROJECT"
```

## DNS64 Configuration

```bash
# DNS64 in GCP is provided automatically by the VPC DNS resolver
# for IPv6-only subnets - no additional configuration needed

# How DNS64 works:
# 1. IPv6-only VM queries: dig AAAA ipv4only.example.com
# 2. GCP DNS64 checks: no AAAA record exists
# 3. GCP DNS64 synthesizes: 64:ff9b::x.x.x.x  (maps IPv4 to NAT64 prefix)
# 4. VM connects to 64:ff9b::1.2.3.4 (IPv4 address 1.2.3.4)
# 5. Cloud NAT64 translates to IPv4 and forwards

# Test DNS64 from IPv6-only VM
gcloud compute ssh vm-ipv6only \
    --project="$PROJECT" \
    --zone=us-east1-b

# Inside VM:
ip -6 addr show                # Only IPv6 address
dig AAAA ipv4only-site.com    # Should return 64:ff9b:: address
ping6 -c 3 64:ff9b::1.1.1.1  # Via NAT64
curl http://ipv4only-site.com  # Automatically uses NAT64
```

## Terraform IPv6-Only Subnet with NAT64

```hcl
# ipv6_only_nat64.tf

variable "project_id" {}
variable "region" { default = "us-east1" }

resource "google_compute_network" "ipv6only" {
  name                    = "vpc-ipv6only"
  auto_create_subnetworks = false
  project                 = var.project_id
}

# IPv6-only subnet
resource "google_compute_subnetwork" "ipv6only" {
  name          = "subnet-ipv6only"
  ip_cidr_range = "10.0.1.0/24"  # Required even for IPv6-only
  region        = var.region
  network       = google_compute_network.ipv6only.id
  project       = var.project_id

  stack_type       = "IPV6_ONLY"
  ipv6_access_type = "INTERNAL"
}

# Cloud Router
resource "google_compute_router" "ipv6only" {
  name    = "router-ipv6only"
  region  = var.region
  network = google_compute_network.ipv6only.id
  project = var.project_id
}

# Cloud NAT providing NAT64
resource "google_compute_router_nat" "nat64" {
  name                               = "nat-nat64"
  router                             = google_compute_router.ipv6only.name
  region                             = var.region
  project                            = var.project_id
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  enable_endpoint_independent_mapping = true
}

# IPv6-only VM
resource "google_compute_instance" "ipv6only" {
  name         = "vm-ipv6only"
  machine_type = "n2-standard-2"
  zone         = "${var.region}-b"
  project      = var.project_id

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.ipv6only.id
    stack_type = "IPV6_ONLY"

    ipv6_access_config {
      network_tier = "PREMIUM"
    }
  }
}
```

## Verify DNS64 and NAT64

```bash
# Inside IPv6-only VM:

# Confirm no IPv4 address
ip addr show | grep "inet "    # Should show nothing or loopback only

# Test DNS64 synthesis
dig AAAA example.com
# Returns: 64:ff9b::5db8:d877 (synthesized from 93.184.216.119)

# Test NAT64 connectivity to IPv4 internet
curl -6 http://example.com/     # Goes through NAT64
wget -6 https://ipv4.google.com/ # NAT64 translates to IPv4

# Test direct IPv6 connectivity (still works)
curl -6 https://ipv6.google.com/
ping6 -c 3 2001:4860:4860::8888
```

## Conclusion

GCP IPv6-only subnets use `stack-type=IPV6_ONLY` and provide DNS64 and NAT64 automatically via the VPC DNS resolver and Cloud NAT. DNS64 synthesizes `64:ff9b::/96` AAAA records for IPv4-only domains, while Cloud NAT64 handles the protocol translation. No additional DNS configuration is needed - GCP's DNS resolver automatically provides DNS64 for IPv6-only subnets. This enables fully IPv6-only workloads while maintaining backward compatibility with IPv4 internet services.
