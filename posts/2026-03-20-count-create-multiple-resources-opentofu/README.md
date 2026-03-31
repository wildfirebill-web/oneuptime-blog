# How to Use count to Create Multiple Resources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Count, Resource, HCL, Infrastructure as Code, DevOps

Description: Learn how to use the count meta-argument to create multiple instances of a resource from a single block in OpenTofu.

---

The `count` meta-argument tells OpenTofu to create multiple instances of a resource. When you set `count = 3`, OpenTofu creates three separate resources all defined by the same block. Each instance is accessed via `count.index` (0, 1, 2...) to differentiate them.

---

## Basic count Usage

```hcl
# Create 3 identical EC2 instances

resource "aws_instance" "web" {
  count = 3

  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  # count.index is 0, 1, 2 for each instance
  tags = {
    Name = "web-server-${count.index + 1}"   # web-server-1, web-server-2, web-server-3
  }
}

# Access individual instances
output "instance_ids" {
  value = aws_instance.web[*].id  # ["i-001", "i-002", "i-003"]
}

output "first_instance_ip" {
  value = aws_instance.web[0].public_ip  # access by index
}
```

---

## Using Variables with count

```hcl
variable "instance_count" {
  type    = number
  default = 2
}

resource "aws_instance" "app" {
  count = var.instance_count

  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  # Use count.index to assign to different subnets
  subnet_id = var.private_subnet_ids[count.index % length(var.private_subnet_ids)]

  tags = {
    Name = "${var.app_name}-${count.index + 1}"
  }
}
```

---

## Conditional Resource Creation with count

```hcl
# Create resource only if a condition is true
variable "enable_monitoring" {
  type    = bool
  default = false
}

resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.enable_monitoring ? 1 : 0   # 1 = create, 0 = don't create

  alarm_name          = "high-cpu-utilization"
  comparison_operator = "GreaterThanThreshold"
  threshold           = 80
  # ...
}

# Reference conditional resource safely
output "alarm_arn" {
  value = var.enable_monitoring ? aws_cloudwatch_metric_alarm.high_cpu[0].arn : null
}
```

---

## count with Lists

```hcl
variable "subnet_cidrs" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "private" {
  count             = length(var.subnet_cidrs)   # count = 3

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.subnet_cidrs[count.index]   # different CIDR per subnet
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "private-subnet-${count.index + 1}"
  }
}
```

---

## Limitations of count

```hcl
# PROBLEM: count creates resources addressed by INDEX [0], [1], [2]
# If you remove an item from the middle of a list, indices shift
# → OpenTofu may destroy and recreate unexpected resources

# Example: remove "us-east-1b" from the middle
variable "azs" {
  default = ["us-east-1a", "us-east-1c"]  # was ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# Previous subnet [1] was in us-east-1b
# After removal, subnet [1] is now in us-east-1c
# OpenTofu plans to MODIFY subnet [1] (potentially destroying it) and destroy subnet [2]
# Use for_each with maps to avoid this problem
```

---

## When to Use count vs for_each

Use `count` when:
- Creating N identical resources
- Creating/not-creating a single optional resource (`count = var.enable ? 1 : 0`)

Use `for_each` when:
- Resources have meaningful identities (names, keys)
- You may add/remove items without affecting other instances

---

## Summary

`count` is perfect for creating N identical instances or conditionally creating/skipping a resource. Use `count.index` to differentiate instances (for names, subnet assignment, etc.). Be aware that count uses integer indices - removing items from the middle of the source list will shift indices and may cause unintended resource replacements. For named resources, prefer `for_each`.
