# How to Set Up VPC Peering with OpenTofu on AWS - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Infrastructure as Code, IaC, VPC Peering, Networking

Description: Learn how to configure AWS VPC peering connections between VPCs in the same or different accounts and regions using OpenTofu.

## Introduction

VPC Peering allows routing traffic between two VPCs privately using AWS infrastructure. This guide covers setting up VPC peering connections, accepting peer requests, and updating route tables using OpenTofu.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with access to both VPCs
- Non-overlapping CIDR blocks

## Step 1: Configure Providers

```hcl
# For same-account peering

provider "aws" {
  region = var.region
  alias  = "requester"
}

# For cross-account peering, use a different provider
provider "aws" {
  region  = var.peer_region
  alias   = "accepter"
  profile = "peer-account-profile"
}
```

## Step 2: Create the Peering Connection

```hcl
# Requester initiates the peering request
resource "aws_vpc_peering_connection" "main" {
  provider    = aws.requester
  vpc_id      = var.requester_vpc_id
  peer_vpc_id = var.accepter_vpc_id
  peer_region = var.peer_region

  # For same-account peering, auto-accept
  auto_accept = var.same_account ? true : false

  tags = {
    Name = "vpc-peer-${var.requester_vpc_id}-${var.accepter_vpc_id}"
    Side = "Requester"
  }
}
```

## Step 3: Accept the Peering Connection (Cross-Account)

```hcl
# Accepter accepts the peering request
resource "aws_vpc_peering_connection_accepter" "peer" {
  provider                  = aws.accepter
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id
  auto_accept               = true

  tags = {
    Name = "vpc-peer-accepter"
    Side = "Accepter"
  }
}
```

## Step 4: Enable DNS Resolution

```hcl
# Enable DNS resolution for requester VPC
resource "aws_vpc_peering_connection_options" "requester" {
  provider                  = aws.requester
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id

  requester {
    allow_remote_vpc_dns_resolution = true
  }
}

# Enable DNS resolution for accepter VPC
resource "aws_vpc_peering_connection_options" "accepter" {
  provider                  = aws.accepter
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id

  accepter {
    allow_remote_vpc_dns_resolution = true
  }
}
```

## Step 5: Update Route Tables

```hcl
# Add route in requester VPC to peer VPC
resource "aws_route" "requester_to_peer" {
  route_table_id            = var.requester_route_table_id
  destination_cidr_block    = var.accepter_vpc_cidr
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id
}

# Add route in accepter VPC to requester VPC
resource "aws_route" "accepter_to_peer" {
  provider                  = aws.accepter
  route_table_id            = var.accepter_route_table_id
  destination_cidr_block    = var.requester_vpc_cidr
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id
}
```

## Step 6: Outputs

```hcl
output "peering_connection_id" {
  value = aws_vpc_peering_connection.main.id
}

output "peering_status" {
  value = aws_vpc_peering_connection.main.accept_status
}
```

## Step 7: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

You have successfully configured VPC peering using OpenTofu. VPC peering provides low-latency, private connectivity between VPCs without requiring a VPN or gateway. Remember that VPC peering is non-transitive - you cannot route traffic through a peered VPC to reach a third VPC. For hub-and-spoke architectures, consider AWS Transit Gateway instead.
