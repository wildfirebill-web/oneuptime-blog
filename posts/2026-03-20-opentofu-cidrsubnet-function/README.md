# How to Use the cidrsubnet Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Networking

Description: Learn how to use the cidrsubnet function in OpenTofu to calculate subnet CIDR blocks programmatically for dynamic network design.

## Introduction

The `cidrsubnet` function in OpenTofu calculates a subnet within a parent CIDR block by adding bits to the prefix length and specifying a network number. It is the most commonly used network function in OpenTofu for dynamic subnet allocation.

## Syntax

```hcl
cidrsubnet(prefix, newbits, netnum)
```

- **prefix** - the parent CIDR block
- **newbits** - the number of additional bits to extend the prefix
- **netnum** - which subnet to select (starting from 0)
- Returns the subnet CIDR

## Basic Examples

```hcl
# VPC: 10.0.0.0/16

# newbits=8 → /24 subnets (16+8=24)

output "first_subnet" {
  value = cidrsubnet("10.0.0.0/16", 8, 0)   # Returns "10.0.0.0/24"
}

output "second_subnet" {
  value = cidrsubnet("10.0.0.0/16", 8, 1)   # Returns "10.0.1.0/24"
}

output "tenth_subnet" {
  value = cidrsubnet("10.0.0.0/16", 8, 10)  # Returns "10.0.10.0/24"
}
```

## Practical Use Cases

### Dynamic Subnet Creation

```hcl
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
}

resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  # Creates: 10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "public-${var.availability_zones[count.index]}"
    Type = "public"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  # Creates: 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "private-${var.availability_zones[count.index]}"
    Type = "private"
  }
}
```

### Tiered Subnet Design

```hcl
locals {
  vpc_cidr = "10.0.0.0/16"

  # Large /20 blocks for tiers (4096 IPs each)
  public_cidr  = cidrsubnet(local.vpc_cidr, 4, 0)   # 10.0.0.0/20
  private_cidr = cidrsubnet(local.vpc_cidr, 4, 1)   # 10.0.16.0/20
  db_cidr      = cidrsubnet(local.vpc_cidr, 4, 2)   # 10.0.32.0/20

  az_count = 3

  # Divide each tier into /24 subnets per AZ
  public_subnets  = [for i in range(local.az_count) : cidrsubnet(local.public_cidr, 4, i)]
  private_subnets = [for i in range(local.az_count) : cidrsubnet(local.private_cidr, 4, i)]
  db_subnets      = [for i in range(local.az_count) : cidrsubnet(local.db_cidr, 4, i)]
}

output "subnet_plan" {
  value = {
    public  = local.public_subnets
    private = local.private_subnets
    db      = local.db_subnets
  }
}
```

### IPv6 Subnetting

```hcl
variable "ipv6_vpc_cidr" {
  type    = string
  default = "2001:db8::/56"  # Example IPv6 range
}

locals {
  ipv6_subnets = [
    for i in range(4) :
    cidrsubnet(var.ipv6_vpc_cidr, 8, i)
  ]
}
```

## Step-by-Step Usage

```bash
tofu console

> cidrsubnet("10.0.0.0/16", 8, 0)
"10.0.0.0/24"
> cidrsubnet("10.0.0.0/16", 8, 5)
"10.0.5.0/24"
> cidrsubnet("10.0.0.0/8", 8, 1)
"10.1.0.0/16"
```

## Understanding newbits and netnum

For `cidrsubnet("10.0.0.0/16", 8, n)`:
- Parent: `/16` (65,536 addresses)
- `newbits = 8` → `/24` subnets (256 addresses each)
- Total possible subnets: `2^8 = 256`
- `netnum 0` → `10.0.0.0/24`
- `netnum 1` → `10.0.1.0/24`
- ...
- `netnum 255` → `10.0.255.0/24`

## Conclusion

The `cidrsubnet` function is the cornerstone of network automation in OpenTofu. It enables fully programmatic subnet design where subnet CIDRs are derived from the parent VPC CIDR using index-based calculation, eliminating the need to hardcode subnet ranges and ensuring all subnets are consistent and non-overlapping.
