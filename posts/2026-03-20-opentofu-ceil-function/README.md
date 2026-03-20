# How to Use the ceil Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the ceil function in OpenTofu to round numbers up to the nearest integer for resource sizing and capacity planning.

## Introduction

The `ceil` function in OpenTofu rounds a number up to the nearest whole integer. This is useful in infrastructure configurations where you need to ensure you always provision enough resources - for example, rounding up the number of nodes needed to handle a given load.

## Syntax

```hcl
ceil(number)
```

- **number** - any numeric value (integer or float)
- Returns the smallest integer greater than or equal to the input.

## Basic Examples

```hcl
output "ceil_positive" {
  value = ceil(1.1)   # Returns 2
}

output "ceil_negative" {
  value = ceil(-1.9)  # Returns -1 (rounds toward zero)
}

output "ceil_whole" {
  value = ceil(5.0)   # Returns 5
}
```

## Practical Use Cases

### Calculating the Number of Instances Needed

When dividing workload across instances, you often get a fractional result. Use `ceil` to always have enough capacity.

```hcl
variable "total_requests_per_second" {
  type    = number
  default = 1500
}

variable "requests_per_instance" {
  type    = number
  default = 400
}

locals {
  # Round up to ensure full coverage
  required_instances = ceil(var.total_requests_per_second / var.requests_per_instance)
}

resource "aws_autoscaling_group" "app" {
  min_size         = local.required_instances
  max_size         = local.required_instances * 2
  desired_capacity = local.required_instances

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  vpc_zone_identifier = var.subnet_ids
}
```

### Disk Capacity Planning

```hcl
variable "data_gb" {
  type    = number
  default = 123.5
}

locals {
  # Always allocate whole GB, rounding up
  allocated_gb = ceil(var.data_gb)
}

resource "aws_ebs_volume" "data" {
  availability_zone = "us-east-1a"
  size              = local.allocated_gb

  tags = {
    Name = "app-data"
  }
}
```

### Sharding Across Availability Zones

```hcl
variable "total_shards" {
  type    = number
  default = 7
}

variable "az_count" {
  type    = number
  default = 3
}

locals {
  # Shards per AZ, rounded up so no shard is left unassigned
  shards_per_az = ceil(var.total_shards / var.az_count)
}

output "shards_per_az" {
  value = local.shards_per_az  # Returns 3
}
```

## Step-by-Step Usage

1. Identify a division or fractional calculation in your config.
2. Wrap the expression with `ceil()` to ensure you always round up.
3. Use `tofu console` to test:

```bash
tofu console

> ceil(2.1)
3
> ceil(2.0)
2
> ceil(-2.1)
-2
```

## Comparison: ceil vs floor

| Expression | Result | Meaning |
|------------|--------|---------|
| `ceil(2.3)` | `3` | Round up |
| `floor(2.3)` | `2` | Round down |
| `ceil(-2.3)` | `-2` | Round toward zero |
| `floor(-2.3)` | `-3` | Round away from zero |

## Best Practices

- Use `ceil` for resource sizing to ensure you never under-provision.
- Use `floor` when you want conservative (smaller) estimates.
- Combine `ceil` with `min` to also apply an upper bound.

## Conclusion

The `ceil` function is essential for capacity planning in OpenTofu. Whenever you compute the number of resources needed through division, `ceil` ensures you always have enough by rounding fractional values up to the next whole integer. This prevents under-provisioning and makes your infrastructure definitions more resilient to varying input sizes.
