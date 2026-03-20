# How to Configure GCP Cloud Interconnect IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, Cloud Interconnect, IPv6, BGP, Dedicated, Partner, Dual-Stack

Description: Configure GCP Cloud Interconnect VLAN attachments for IPv6 BGP routing between your data center and GCP.

## Introduction

GCP Cloud Interconnect IPv6 enables private IPv6 connectivity between cloud resources and on-premises or inter-VPC networks. Proper configuration requires setting up dual-stack support, IPv6 BGP sessions, and route advertisement.

## Prerequisites

- VPC/VNet with dual-stack (IPv4 + IPv6) subnets
- An existing GCP account with appropriate IAM permissions
- IPv6 address space allocated for the connection

## Step 1: Verify IPv6 Prerequisites

```bash
# Check VPC has IPv6 CIDR

gcloud compute networks subnets list --filter='ipv6AccessType=INTERNAL OR ipv6AccessType=EXTERNAL'
```

## Step 2: Enable IPv6 on the Service

```bash
# Enable IPv6 on subnet
gcloud compute networks subnets update my-subnet     --region us-central1     --stack-type IPV4_IPV6     --ipv6-access-type INTERNAL

# Verify
gcloud compute networks subnets describe my-subnet     --region us-central1     --format="value(ipv6CidrRange)"
```

## Step 3: Configure IPv6 BGP

```bash
# Configure BGP for IPv6 on Cloud Router
gcloud compute routers create my-router     --region us-central1     --network my-network     --asn 65000

# Add IPv6 BGP peer
gcloud compute routers add-bgp-peer my-router     --region us-central1     --peer-name ipv6-peer     --peer-asn 65001     --peer-ip-address "2001:db8::2"     --interface my-interface     --address "2001:db8::1"
```

## Step 4: Add IPv6 Routes

```bash
# Advertise IPv6 prefix from Cloud Router
gcloud compute routers update-bgp-peer my-router \
    --region us-central1 \
    --peer-name ipv6-peer \
    --advertisement-mode custom \
    --set-advertisement-ranges '2001:db8::/48'
```

## Step 5: Test IPv6 Connectivity

```bash
# Test from cloud instance
ping6 -c 3 <on-premises-ipv6-address>

# Verify route is learned
gcloud compute routers get-status my-router --region us-central1 | grep ipv6
```

## Step 6: Terraform Example

```hcl
# Terraform for GCP Cloud Interconnect IPv6
resource "google_compute_router_peer" "ipv6_peer" {
  name            = "ipv6-bgp-peer"
  router          = google_compute_router.main.name
  region          = var.region
  peer_asn        = 65001
  peer_ip_address = "2001:db8::2"
  interface       = google_compute_router_interface.main.name
}
```

## Conclusion

GCP Cloud Interconnect IPv6 requires enabling dual-stack at the subnet level, configuring IPv6 BGP sessions, and adding IPv6 routes in the relevant route tables. Test connectivity end-to-end after configuration. Use Terraform for declarative, repeatable deployments. Monitor IPv6 BGP session state and route advertisement with OneUptime's network health checks.
