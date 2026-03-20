# How to Create Subnets with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Subnets, Networking, Infrastructure as Code

Description: Learn how to create public, private, and database subnets in AWS with OpenTofu using dynamic CIDR allocation and availability zone distribution.

## Introduction

Subnets divide your VPC CIDR block into smaller networks assigned to specific availability zones. A well-designed subnet structure separates public-facing resources, application servers, and databases into distinct tiers—each with its own routing and security posture.

## Basic Subnet Creation

```hcl
resource "aws_subnet" "public_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  # Automatically assign public IPs to instances in this subnet
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-1"
    Tier = "public"
  }
}
```

## Dynamic Subnet Creation with `count`

Use `count` to create multiple subnets programmatically:

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

# Public subnets — one per AZ
resource "aws_subnet" "public" {
  count = length(data.aws_availability_zones.available.names)

  vpc_id            = aws_vpc.main.id
  # cidrsubnet(base, newbits, netnum) carves subnets from the VPC CIDR
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name = "public-${data.aws_availability_zones.available.names[count.index]}"
    Tier = "public"
    # Tag for EKS ALB auto-discovery
    "kubernetes.io/role/elb" = "1"
  }
}
```

## Three-Tier Subnet Architecture

A common pattern is three subnet tiers: public, private (application), and database:

```hcl
locals {
  az_count = min(var.max_az_count, length(data.aws_availability_zones.available.names))
}

# Tier 1: Public subnets (load balancers, NAT gateways, bastion hosts)
resource "aws_subnet" "public" {
  count             = local.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "public-${count.index + 1}", Tier = "public" }
}

# Tier 2: Private subnets (application servers, ECS tasks, Lambda)
resource "aws_subnet" "private" {
  count             = local.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index + local.az_count)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "private-${count.index + 1}", Tier = "private" }
}

# Tier 3: Database subnets (RDS, ElastiCache)
resource "aws_subnet" "database" {
  count             = local.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index + local.az_count * 2)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "database-${count.index + 1}", Tier = "database" }
}

# RDS subnet group using the database subnets
resource "aws_db_subnet_group" "main" {
  name       = "${var.name}-db-subnet-group"
  subnet_ids = aws_subnet.database[*].id

  tags = {
    Name = "${var.name}-db-subnet-group"
  }
}
```

## Subnet CIDR Visualisation

With `vpc_cidr = "10.0.0.0/16"`, `max_az_count = 2`, and `/4` offset (`newbits = 4`):

| Subnet | CIDR | Tier |
|---|---|---|
| public-1 | 10.0.0.0/20 | Public |
| public-2 | 10.0.16.0/20 | Public |
| private-1 | 10.0.32.0/20 | Private |
| private-2 | 10.0.48.0/20 | Private |
| database-1 | 10.0.64.0/20 | Database |
| database-2 | 10.0.80.0/20 | Database |

## Outputs

```hcl
output "public_subnet_ids"   { value = aws_subnet.public[*].id }
output "private_subnet_ids"  { value = aws_subnet.private[*].id }
output "database_subnet_ids" { value = aws_subnet.database[*].id }
output "db_subnet_group_name" { value = aws_db_subnet_group.main.name }
```

## Conclusion

Dynamic subnet creation with `count` and `cidrsubnet` makes multi-AZ, multi-tier networking configurations compact and maintainable. Tag subnets correctly for EKS, ELB, and other AWS service auto-discovery, and always separate database subnets from application subnets for defence in depth.
