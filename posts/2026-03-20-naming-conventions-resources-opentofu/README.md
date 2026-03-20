# How to Follow Naming Conventions for Resources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Naming Conventions, Best Practices, Infrastructure as Code, Organizations, DevOps

Description: Learn how to establish consistent naming conventions for OpenTofu resource blocks and cloud resource names to improve readability, searchability, and team collaboration.

## Introduction

Naming conventions in OpenTofu serve two purposes: naming the OpenTofu resource block (the logical name in your `.tf` files) and naming the actual cloud resource (the `name` or `Name` tag). Consistency in both makes infrastructure self-documenting.

## OpenTofu Resource Block Naming

Resource block names (the identifier after the resource type) follow these conventions:

```hcl
# Pattern: <purpose>_<qualifier>

# Use snake_case (underscores)
# Be descriptive but concise

# GOOD
resource "aws_vpc" "main" {}
resource "aws_vpc" "management" {}
resource "aws_subnet" "private" {}
resource "aws_subnet" "public_1" {}
resource "aws_security_group" "web" {}
resource "aws_security_group" "db" {}
resource "aws_instance" "bastion" {}

# AVOID
resource "aws_vpc" "my_vpc" {}           # Redundant "my_"
resource "aws_subnet" "subnet1" {}       # Not descriptive
resource "aws_instance" "EC2Instance" {} # CamelCase - use snake_case
```

## Cloud Resource Naming Convention

Cloud resource names should encode: company/project, environment, region (optional), and purpose:

```hcl
# Pattern: {company}-{environment}-{region?}-{purpose}
# or:      {project}-{environment}-{purpose}

locals {
  name_prefix = "${var.company}-${var.environment}"
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = {
    Name = "${local.name_prefix}-vpc"   # e.g., "acme-prod-vpc"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = {
    # e.g., "acme-prod-private-a", "acme-prod-private-b"
    Name = "${local.name_prefix}-private-${substr(data.aws_availability_zones.available.names[count.index], -1, 1)}"
  }
}
```

## IAM Resource Naming

```hcl
# IAM roles: {company}-{service}-{purpose}-role
resource "aws_iam_role" "eks_node" {
  name = "${local.name_prefix}-eks-node-role"
}

# IAM policies: {company}-{purpose}-policy
resource "aws_iam_policy" "s3_read" {
  name = "${local.name_prefix}-s3-read-policy"
}

# IAM instance profiles: {company}-{service}-profile
resource "aws_iam_instance_profile" "ec2_app" {
  name = "${local.name_prefix}-ec2-app-profile"
  role = aws_iam_role.ec2_app.name
}
```

## S3 Bucket Naming

```hcl
# S3: globally unique, lowercase, no underscores
resource "aws_s3_bucket" "app_data" {
  # Pattern: {company}-{environment}-{purpose}-{account_id_suffix}
  bucket = "${var.company}-${var.environment}-app-data-${substr(data.aws_caller_identity.current.account_id, -4, 4)}"
}
```

## Using Variables for Naming Consistency

```hcl
# variables.tf - centralize naming inputs
variable "company" {
  type        = string
  description = "Company name abbreviation for resource naming"
  default     = "acme"
}

variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "project" {
  type = string
}

locals {
  # Consistent prefix used in all resource names
  name_prefix = "${var.company}-${var.project}-${var.environment}"
  common_tags = {
    Company     = var.company
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "opentofu"
  }
}
```

## Conclusion

Consistent naming conventions make infrastructure self-documenting. Use snake_case for OpenTofu resource block names, encode environment and purpose in cloud resource names, and centralize naming logic in a `locals` block so all resources follow the same pattern automatically.
