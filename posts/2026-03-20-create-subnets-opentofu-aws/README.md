# How to Create VPC Subnets with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, VPC, Subnets, Networking, Infrastructure as Code

Description: Learn how to create public and private subnets across multiple availability zones in AWS using OpenTofu.

---

Subnets divide a VPC's IP address space into segments that can be placed in specific availability zones. OpenTofu makes it easy to create multiple subnets with dynamic CIDR calculation and AZ assignment.

---

## Create Public Subnets

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-${count.index + 1}"
    Type = "public"
  }
}
```

---

## Create Private Subnets

```hcl
resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "private-subnet-${count.index + 1}"
    Type = "private"
  }
}
```

---

## Create Database Subnets

```hcl
resource "aws_subnet" "database" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 20)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "database-subnet-${count.index + 1}"
    Type = "database"
  }
}
```

---

## Create Route Tables and Associate

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "public-rt" }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

---

## Output Subnet IDs

```hcl
output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

---

## Summary

Use `cidrsubnet(var.vpc_cidr, 8, index)` to calculate CIDR blocks dynamically and `data.aws_availability_zones.available.names[count.index]` to spread subnets across AZs. Set `map_public_ip_on_launch = true` for public subnets. Associate subnets with route tables using `aws_route_table_association` to control traffic routing.
