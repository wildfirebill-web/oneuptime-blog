# How to Use count to Create Multiple Resources in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Resources, Count, Infrastructure as Code, DevOps

Description: A guide to using the count meta-argument in OpenTofu to create multiple instances of a resource from a single block.

## Introduction

The `count` meta-argument creates multiple instances of a resource from a single resource block. Instead of writing separate blocks for each instance, you declare a resource once with `count = N` and OpenTofu creates N identical (or slightly varied) resources.

## Basic count Usage

```hcl
# Create 3 identical EC2 instances

resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # Use count.index for differentiation
  tags = {
    Name  = "web-server-${count.index}"  # web-server-0, web-server-1, web-server-2
    Index = count.index
  }
}
```

## count.index

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index)  # 10.0.0.0/24, 10.0.1.0/24, etc.
  availability_zone = var.availability_zones[count.index]  # AZ from list by index

  tags = {
    Name = "public-subnet-${count.index + 1}"  # Start from 1 for readability
  }
}
```

## Conditional Resource with count

```hcl
variable "create_bastion" {
  type    = bool
  default = false
}

# count = 0 means don't create, count = 1 means create
resource "aws_instance" "bastion" {
  count = var.create_bastion ? 1 : 0

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "bastion-host"
  }
}

# Accessing conditional resource (index 0 or use [*] for list)
output "bastion_ip" {
  value = var.create_bastion ? aws_instance.bastion[0].public_ip : null
}
```

## Referencing count Resources

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# Access specific instance
output "first_web_ip" {
  value = aws_instance.web[0].public_ip  # Index 0
}

# Access all instances (splat expression)
output "all_web_ips" {
  value = aws_instance.web[*].public_ip  # Returns list
}

# Use in another resource
resource "aws_lb_target_group_attachment" "web" {
  count            = length(aws_instance.web)  # Same count
  target_group_arn = aws_lb_target_group.web.arn
  target_id        = aws_instance.web[count.index].id
  port             = 80
}
```

## count with List Variables

```hcl
variable "server_names" {
  type    = list(string)
  default = ["web-1", "web-2", "web-3"]
}

resource "aws_instance" "server" {
  count         = length(var.server_names)
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = var.server_names[count.index]  # Name from list
  }
}
```

## Limitations of count

```hcl
# PROBLEM: Inserting an element in the middle shifts indices
# If server_names = ["a", "b", "c"] becomes ["a", "x", "b", "c"]
# "b" becomes index 2, "c" becomes index 3
# OpenTofu thinks it needs to destroy "b" and create "x", rename "b" and "c"
# This causes unnecessary resource recreation

# SOLUTION: Use for_each with a unique identifier instead of count
# When order matters: count is fine
# When items have unique identities: use for_each
```

## count in State

```bash
# Resources created with count use index in state:
tofu state list
# aws_instance.web[0]
# aws_instance.web[1]
# aws_instance.web[2]

# Access specific indexed resource
tofu state show 'aws_instance.web[0]'
```

## Conclusion

The `count` meta-argument is the simplest way to create multiple identical resources. It's best suited for situations where resources are interchangeable and their order doesn't matter (like subnets across AZs). For scenarios where resources have unique identities or when items may be inserted/removed from the middle of a list, `for_each` provides more stable resource addressing. The `count = bool ? 1 : 0` pattern for conditional creation is idiomatic OpenTofu.
