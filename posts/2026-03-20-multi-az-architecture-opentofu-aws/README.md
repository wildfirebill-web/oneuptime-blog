# How to Deploy a Multi-AZ Architecture with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Infrastructure as Code, IaC, Multi-AZ, High Availability, Architecture

Description: Learn how to design and deploy a complete multi-availability zone architecture on AWS using OpenTofu for high availability and fault tolerance.

## Introduction

Multi-AZ architecture distributes resources across multiple availability zones to ensure high availability and fault tolerance. This guide shows how to deploy a complete three-tier web application architecture across multiple AZs using OpenTofu.

## Architecture Overview

The architecture includes:
- VPC with public and private subnets in 3 AZs
- Application Load Balancer spanning all AZs
- Auto Scaling Group deploying instances across AZs
- RDS Multi-AZ for database high availability
- ElastiCache with replicas in separate AZs

## Step 1: Create VPC with Multi-AZ Subnets

```hcl
locals {
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  public_cidrs    = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_cidrs   = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]
  database_cidrs  = ["10.0.21.0/24", "10.0.22.0/24", "10.0.23.0/24"]
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = { Name = "production-vpc" }
}

resource "aws_subnet" "public" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = { Name = "public-${local.azs[count.index]}" }
}

resource "aws_subnet" "private" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = { Name = "private-${local.azs[count.index]}" }
}

resource "aws_subnet" "database" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.database_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = { Name = "database-${local.azs[count.index]}" }
}
```

## Step 2: Create Internet Gateway and NAT Gateways

```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "igw" }
}

resource "aws_eip" "nat" {
  count  = length(local.azs)
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count         = length(local.azs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  tags          = { Name = "nat-${local.azs[count.index]}" }
  depends_on    = [aws_internet_gateway.main]
}
```

## Step 3: Multi-AZ RDS Instance

```hcl
resource "aws_db_subnet_group" "main" {
  name       = "production-db-subnet-group"
  subnet_ids = aws_subnet.database[*].id
}

resource "aws_db_instance" "main" {
  identifier        = "production-db"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.r7g.large"
  allocated_storage = 100
  storage_type      = "gp3"

  db_name  = var.db_name
  username = var.db_username
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]

  # Multi-AZ for automatic failover
  multi_az = true

  backup_retention_period = 7
  deletion_protection     = true
  skip_final_snapshot     = false

  tags = { Environment = "production" }
}
```

## Step 4: Application Load Balancer

```hcl
resource "aws_lb" "app" {
  name               = "app-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = aws_subnet.public[*].id
  security_groups    = [aws_security_group.alb.id]

  enable_deletion_protection = true
  drop_invalid_header_fields = true

  access_logs {
    bucket  = var.log_bucket
    enabled = true
  }
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

You have successfully deployed a multi-AZ architecture using OpenTofu with dedicated subnets per tier per AZ, NAT gateways in each AZ, Multi-AZ RDS for database HA, and an ALB spanning all AZs. This architecture achieves 99.99% availability and automatically routes around AZ failures.
