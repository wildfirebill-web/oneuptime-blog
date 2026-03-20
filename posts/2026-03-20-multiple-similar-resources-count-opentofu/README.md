# How to Create Multiple Similar Resources with count in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Count, HCL, Resource Creation, Scaling

Description: Learn how to use the count meta-argument in OpenTofu to create multiple instances of a resource, using count.index for unique naming and configuration.

## Introduction

The `count` meta-argument creates multiple instances of a resource. Each instance is identified by its zero-based index (`count.index`), which you can use to generate unique names, select different subnets, or vary configuration across instances.

## Creating Multiple EC2 Instances

```hcl
variable "instance_count" {
  description = "Number of application servers to create"
  type        = number
  default     = 3
}

variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_instance" "app" {
  count = var.instance_count

  ami           = data.aws_ami.app.id
  instance_type = "t3.medium"

  # Distribute instances across AZs using modulo
  subnet_id = var.private_subnet_ids[count.index % length(var.private_subnet_ids)]

  tags = {
    Name  = "app-server-${count.index + 1}"  # 1-based human-friendly names
    Index = count.index
    Role  = "app"
  }
}

# Create an EIP for each instance

resource "aws_eip" "app" {
  count    = var.instance_count
  instance = aws_instance.app[count.index].id
  domain   = "vpc"
}

output "instance_ips" {
  value = aws_eip.app[*].public_ip
}
```

## Creating Multiple Subnets Across AZs

```hcl
variable "vpc_id"             { type = string }
variable "availability_zones" { type = list(string) }

variable "private_subnet_cidrs" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id            = var.vpc_id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "private-subnet-${var.availability_zones[count.index]}"
    Tier = "private"
  }
}

resource "aws_nat_gateway" "per_az" {
  count         = length(var.availability_zones)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = { Name = "nat-${var.availability_zones[count.index]}" }
}

resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"
}
```

## Count with Splat Expressions

Use splat expressions (`[*]`) to collect attributes from all instances.

```hcl
resource "aws_instance" "workers" {
  count         = var.worker_count
  ami           = data.aws_ami.worker.id
  instance_type = "c5.large"
  subnet_id     = var.private_subnet_ids[count.index % length(var.private_subnet_ids)]

  tags = { Name = "worker-${count.index + 1}" }
}

output "worker_ids" {
  # Collect all instance IDs using splat
  value = aws_instance.workers[*].id
}

output "worker_private_ips" {
  value = aws_instance.workers[*].private_ip
}

# Pass all worker IPs to a load balancer target group
resource "aws_lb_target_group_attachment" "workers" {
  count            = var.worker_count
  target_group_arn = aws_lb_target_group.workers.arn
  target_id        = aws_instance.workers[count.index].id
  port             = 8080
}
```

## Limitations: count vs for_each

```hcl
# PROBLEM: count-based resources are identified by index
# If you remove index 1 from ["a", "b", "c"], OpenTofu replans "b" and "c"
resource "aws_s3_bucket" "by_count" {
  count  = length(var.bucket_names)
  bucket = var.bucket_names[count.index]
  # Removing any name causes re-indexing and potential bucket replacement!
}

# BETTER: use for_each when items have stable identifiers
resource "aws_s3_bucket" "by_name" {
  for_each = toset(var.bucket_names)
  bucket   = each.key
  # Each bucket is identified by its name, not position
}
```

## Conclusion

`count` is ideal when you need N identical or near-identical resources where the index is meaningful (e.g., distributing across AZs). Use splat expressions to collect outputs from all instances. However, for named resources where items may be added or removed, prefer `for_each` to avoid unintended resource replacement from index shifting.
