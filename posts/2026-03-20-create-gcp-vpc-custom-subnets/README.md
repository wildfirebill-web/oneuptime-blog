# How to Create a GCP VPC Network with Custom IPv4 Subnets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, VPC, IPv4, Subnets, Networking, Cloud

Description: Create a GCP Virtual Private Cloud network with custom mode subnets and specific IPv4 CIDR ranges using the gcloud CLI, and understand GCP's global VPC model.

## Introduction

GCP VPC networks are global - a single VPC spans all regions. Subnets are regional, meaning each subnet is associated with one region but hosts can communicate globally across subnets. Use custom mode VPC (not auto mode) to control your IPv4 address space precisely.

## Creating a Custom Mode VPC

```bash
PROJECT_ID="my-gcp-project"

# Create a custom VPC (no subnets created automatically)

gcloud compute networks create prod-vpc \
  --project=$PROJECT_ID \
  --subnet-mode=custom \
  --bgp-routing-mode=global \
  --mtu=1460
```

`--subnet-mode=custom` prevents GCP from automatically creating subnets in every region. `--bgp-routing-mode=global` enables Cloud Router routes to be advertised globally across regions.

## Creating Subnets

```bash
# Web tier subnet in us-central1
gcloud compute networks subnets create web-subnet \
  --project=$PROJECT_ID \
  --network=prod-vpc \
  --region=us-central1 \
  --range=10.1.1.0/24

# App tier subnet in us-central1
gcloud compute networks subnets create app-subnet \
  --project=$PROJECT_ID \
  --network=prod-vpc \
  --region=us-central1 \
  --range=10.1.2.0/24

# Database subnet in us-central1
gcloud compute networks subnets create db-subnet \
  --project=$PROJECT_ID \
  --network=prod-vpc \
  --region=us-central1 \
  --range=10.1.3.0/24

# Second region subnet in us-east1
gcloud compute networks subnets create web-subnet-east \
  --project=$PROJECT_ID \
  --network=prod-vpc \
  --region=us-east1 \
  --range=10.2.1.0/24
```

## Enabling Private Google Access on a Subnet

Allows VMs without public IPs to access Google APIs:

```bash
gcloud compute networks subnets update app-subnet \
  --project=$PROJECT_ID \
  --region=us-central1 \
  --enable-private-ip-google-access
```

## Listing VPCs and Subnets

```bash
# List all VPC networks
gcloud compute networks list --project=$PROJECT_ID

# List all subnets
gcloud compute networks subnets list \
  --project=$PROJECT_ID \
  --filter="network:prod-vpc"

# Describe a subnet to see its range
gcloud compute networks subnets describe web-subnet \
  --project=$PROJECT_ID \
  --region=us-central1
```

## Expanding a Subnet's IPv4 Range

GCP allows expanding (but not shrinking) a subnet's CIDR:

```bash
# Expand from /24 to /23 (must be a superset of current range)
gcloud compute networks subnets expand-ip-range web-subnet \
  --project=$PROJECT_ID \
  --region=us-central1 \
  --prefix-length=23
```

## GCP VPC vs Other Clouds

| Feature | GCP | AWS | Azure |
|---|---|---|---|
| VPC scope | Global | Regional | Regional |
| Subnet scope | Regional | AZ-level | Regional |
| Auto-mode | Optional | N/A | N/A |
| Default route | Via default-internet-gateway | Via IGW | Via internet |

## Deleting a Subnet

```bash
gcloud compute networks subnets delete web-subnet \
  --project=$PROJECT_ID \
  --region=us-central1 \
  --quiet
```

## Conclusion

Create GCP VPCs with `--subnet-mode=custom` to control your address space. Add subnets per region with `gcloud compute networks subnets create`. Enable Private Google Access on subnets where VMs need to reach Google APIs without public IPs. GCP's global VPC model means one VPC can serve all regions with consistent firewall rules and routing.
