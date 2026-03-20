# How to Create a NAT Gateway with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Infrastructure as Code, IaC, NAT Gateway, Networking

Description: Learn how to create AWS NAT Gateways for private subnet internet access with elastic IPs and route tables using OpenTofu.

## Introduction

A NAT Gateway enables instances in private subnets to initiate outbound connections to the internet while preventing inbound connections. This guide covers deploying highly available NAT Gateways using OpenTofu.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials configured
- Existing VPC with public and private subnets

## Step 1: Configure the Provider

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

## Step 2: Create Elastic IPs for NAT Gateways

```hcl
# One EIP per AZ for high availability
resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"

  tags = {
    Name = "nat-eip-${var.availability_zones[count.index]}"
  }
}
```

## Step 3: Create NAT Gateways

```hcl
# One NAT Gateway per AZ for high availability
resource "aws_nat_gateway" "main" {
  count         = length(var.availability_zones)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = var.public_subnet_ids[count.index]  # Must be in public subnet

  tags = {
    Name = "nat-gateway-${var.availability_zones[count.index]}"
  }

}
# Note: Add depends_on = [aws_internet_gateway.main] if the IGW is defined in the same module.
```

## Step 4: Create Private Route Tables

```hcl
# Route table per AZ pointing to its NAT gateway
resource "aws_route_table" "private" {
  count  = length(var.availability_zones)
  vpc_id = var.vpc_id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "private-rt-${var.availability_zones[count.index]}"
  }
}

# Associate private subnets with their route tables
resource "aws_route_table_association" "private" {
  count          = length(var.private_subnet_ids)
  subnet_id      = var.private_subnet_ids[count.index]
  route_table_id = aws_route_table.private[count.index].id
}
```

## Step 5: Cost Optimization - Single NAT Gateway

```hcl
# For non-production: single NAT Gateway to save costs
resource "aws_nat_gateway" "single" {
  count         = var.environment == "production" ? 0 : 1
  allocation_id = aws_eip.nat[0].id
  subnet_id     = var.public_subnet_ids[0]

  tags = {
    Name = "nat-gateway-single"
  }
}
```

## Step 6: Outputs

```hcl
output "nat_gateway_ids" {
  value = aws_nat_gateway.main[*].id
}

output "nat_public_ips" {
  value = aws_eip.nat[*].public_ip
}
```

## Step 7: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

You have successfully created NAT Gateways using OpenTofu with one gateway per availability zone for high availability. For production workloads, always deploy a NAT Gateway per AZ to avoid cross-AZ data transfer costs and ensure resilience. For development environments, a single NAT Gateway reduces costs significantly.
