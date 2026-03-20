# How to Plan IPv4 CIDR Allocation for Multi-Region GCP Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IPv4, CIDR, Planning, Multi-Region, VPC Design

Description: Design a systematic IPv4 CIDR allocation strategy for multi-region GCP deployments, accounting for subnets, secondary ranges, VPC peering, and future expansion.

## Introduction

GCP's global VPC requires careful CIDR planning because all subnets in all regions share one VPC address space. Poor planning leads to overlapping ranges that break VPC peering, Shared VPC, and hybrid connectivity. A structured approach prevents address exhaustion and enables seamless expansion.

## GCP CIDR Planning Principles

1. **Reserve blocks per region** — allocate a supernet per region so regional subnets stay contiguous
2. **Reserve secondary ranges** — GKE pods and services need dedicated secondary ranges
3. **Avoid RFC 1918 overlap** — don't use ranges already used by on-premises or peered networks
4. **Plan for peering** — peered VPCs cannot have overlapping ranges

## Multi-Region CIDR Allocation Example

```
prod-vpc: 10.0.0.0/8 (total address space)

├── us-central1:     10.1.0.0/16
│   ├── web-subnet:      10.1.1.0/24
│   ├── app-subnet:      10.1.2.0/24
│   ├── db-subnet:       10.1.3.0/24
│   ├── gke-nodes:       10.1.10.0/22
│   ├── gke-pods:        10.1.64.0/18  (secondary)
│   └── gke-services:    10.1.128.0/20 (secondary)
│
├── us-east1:        10.2.0.0/16
│   ├── web-subnet:      10.2.1.0/24
│   ├── app-subnet:      10.2.2.0/24
│   ├── gke-nodes:       10.2.10.0/22
│   └── gke-pods:        10.2.64.0/18
│
└── europe-west1:    10.3.0.0/16
    ├── web-subnet:      10.3.1.0/24
    └── app-subnet:      10.3.2.0/24
```

## Creating Subnets from the Plan

```bash
PROJECT_ID="my-gcp-project"

# us-central1 subnets
gcloud compute networks subnets create web-uscentral \
  --project=$PROJECT_ID \
  --network=prod-vpc \
  --region=us-central1 \
  --range=10.1.1.0/24

gcloud compute networks subnets create app-uscentral \
  --project=$PROJECT_ID \
  --network=prod-vpc \
  --region=us-central1 \
  --range=10.1.2.0/24 \
  --secondary-range gke-pods=10.1.64.0/18,gke-services=10.1.128.0/20

# us-east1 subnets
gcloud compute networks subnets create web-useast \
  --project=$PROJECT_ID \
  --network=prod-vpc \
  --region=us-east1 \
  --range=10.2.1.0/24

# europe-west1 subnets
gcloud compute networks subnets create web-euwest \
  --project=$PROJECT_ID \
  --network=prod-vpc \
  --region=europe-west1 \
  --range=10.3.1.0/24
```

## VPC Peering Considerations

If you peer with a dev or staging VPC, use separate supernets:

```
prod-vpc:    10.0.0.0/8
staging-vpc: 172.16.0.0/12
dev-vpc:     192.168.0.0/16
on-premises: 10.200.0.0/16 (must not overlap with prod-vpc)
```

## GKE Secondary Range Sizing

| Cluster Size | Pod CIDR | Service CIDR |
|---|---|---|
| Small (<110 pods/node, <100 nodes) | /19 (8,192 IPs) | /22 (1,024 IPs) |
| Medium (<110 pods/node, <1000 nodes) | /14 (262,144 IPs) | /20 (4,096 IPs) |
| Large | /11 | /18 |

## Checking for CIDR Conflicts

```bash
# List all subnets and their ranges
gcloud compute networks subnets list \
  --project=$PROJECT_ID \
  --format="table(name, region, ipCidrRange, secondaryIpRanges.list())"
```

## Documenting the CIDR Plan

Maintain a CIDR registry:

```yaml
# cidr-registry.yaml
vpc:
  name: prod-vpc
  address_space: 10.0.0.0/8

regions:
  us-central1:
    supernet: 10.1.0.0/16
    subnets:
      - name: web-uscentral
        cidr: 10.1.1.0/24
        purpose: web-tier
      - name: app-uscentral
        cidr: 10.1.2.0/24
        purpose: app-tier
        secondary_ranges:
          gke-pods: 10.1.64.0/18
          gke-services: 10.1.128.0/20
```

## Conclusion

Allocate one supernet (e.g., /16) per region within a larger VPC address space. Reserve secondary ranges for GKE pod and service IPs. Document your CIDR registry before creating any subnets. Ensure peered VPCs use completely separate supernets to avoid conflicts when expanding. Use `10.0.0.0/8` for prod and distinct RFC 1918 ranges for staging and dev environments.
