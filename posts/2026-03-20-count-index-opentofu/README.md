# How to Use count and count.index in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Count, Count.index, HCL, Loop, Infrastructure as Code

Description: Learn how to use count and count.index in OpenTofu to create multiple instances of a resource - including patterns for naming, subnet allocation, and when to use for_each instead.

## Introduction

`count` creates N copies of a resource. `count.index` gives you the current iteration number (0-based). Together, they enable creating subnets across availability zones, numbered instances, and repeated resources. However, `for_each` is often the better choice for production use.

## Basic count Usage

```hcl
variable "instance_count" {
  type    = number
  default = 3
}

resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.medium"

  tags = {
    Name = "web-server-${count.index + 1}"  # web-server-1, web-server-2, web-server-3
  }
}

# Reference specific instances

output "first_instance_ip"  { value = aws_instance.web[0].private_ip }
output "all_instance_ips"   { value = aws_instance.web[*].private_ip }
```

## Subnets Across Availability Zones

A common pattern - create subnets in multiple AZs:

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "public" {
  count = 3

  vpc_id            = aws_vpc.main.id
  availability_zone = data.aws_availability_zones.available.names[count.index]

  # Allocate sequential CIDRs: 10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24
  cidr_block = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 1)

  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-${data.aws_availability_zones.available.names[count.index]}"
    Type = "public"
  }
}

resource "aws_subnet" "private" {
  count = 3

  vpc_id            = aws_vpc.main.id
  availability_zone = data.aws_availability_zones.available.names[count.index]

  # Private subnets: 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24
  cidr_block = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 10)

  tags = {
    Name = "private-subnet-${data.aws_availability_zones.available.names[count.index]}"
    Type = "private"
  }
}
```

## Using count with Lists

Pair `count.index` with a list variable to use different values per instance:

```hcl
variable "availability_zones" {
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "public_cidrs" {
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  availability_zone = var.availability_zones[count.index]
  cidr_block        = var.public_cidrs[count.index]

  tags = { Name = "public-${var.availability_zones[count.index]}" }
}
```

## count for Optional Resources

```hcl
variable "enable_nat_gateway" {
  type    = bool
  default = true
}

resource "aws_nat_gateway" "main" {
  count         = var.enable_nat_gateway ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id

  tags = { Name = "main-nat" }
}

resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? 1 : 0
  domain = "vpc"
}
```

## count vs for_each: When to Use Which

```hcl
# USE count WHEN:
# - Creating N identical or sequentially-numbered resources
# - Creating an optional resource (count = 0 or 1)
# - Resources are identified by index (list semantics)

# USE for_each WHEN:
# - Resources are identified by a meaningful key (map semantics)
# - Removing one item shouldn't affect other resources
# - Resources need stable state addresses

# EXAMPLE: Why for_each is usually better for named resources
# With count: if you remove "api" from the middle, "db" shifts from index 2 to 1
#             OpenTofu will destroy and recreate "db"
variable "services" {
  default = ["web", "api", "db"]
}

# FRAGILE: index-based, removal causes cascading changes
resource "aws_ecs_service" "bad" {
  count = length(var.services)
  name  = var.services[count.index]
}

# ROBUST: key-based, removal only affects that one service
resource "aws_ecs_service" "good" {
  for_each = toset(var.services)
  name     = each.value
}
```

## Collecting Outputs from count Resources

```hcl
# Collect all public subnet IDs
output "public_subnet_ids" {
  value = aws_subnet.public[*].id
  # ["subnet-abc", "subnet-def", "subnet-ghi"]
}

# Collect with for expression (more flexibility)
output "subnet_info" {
  value = [for s in aws_subnet.public : {
    id   = s.id
    cidr = s.cidr_block
    az   = s.availability_zone
  }]
}
```

## Conclusion

`count` and `count.index` are the right tool for creating multiple sequential or identical resources, optional resources (`count = 0/1`), and subnet/AZ distribution patterns where the index correlates with list positions. For production workloads where resources have meaningful names (services, users, environments), prefer `for_each` - it provides stable state addresses that don't cascade changes when items are removed from the middle of a list.
