# How to Use the min Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the min function in OpenTofu to return the smallest value from a set of numbers for capping resource sizes and enforcing upper limits.

## Introduction

The `min` function in OpenTofu returns the smallest value from a given set of numbers. It is particularly useful for capping resource sizes, enforcing upper limits, and preventing over-provisioning in your infrastructure configurations.

## Syntax

```hcl
min(number, number, ...)
```

- Accepts two or more numeric arguments.
- Returns the smallest among them.
- Works with a list using the splat operator: `min(list...)`

## Basic Examples

```hcl
output "simple_min" {
  value = min(5, 2, 8, 1)  # Returns 1
}

output "two_values" {
  value = min(10, 20)  # Returns 10
}

output "negative_values" {
  value = min(-5, -3, -10)  # Returns -10
}
```

## Practical Use Cases

### Capping Instance Count to a Maximum

```hcl
variable "desired_instances" {
  type    = number
  default = 50
}

locals {
  # Never exceed the hard cap of 20 instances
  instance_count = min(var.desired_instances, 20)
}

resource "aws_autoscaling_group" "app" {
  desired_capacity    = local.instance_count
  max_size            = 20
  min_size            = 1
  vpc_zone_identifier = var.subnet_ids

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
}
```

### Limiting Replica Count per Environment

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

variable "requested_replicas" {
  type    = number
  default = 10
}

locals {
  # In dev, cap replicas at 2 to save costs
  max_replicas   = var.environment == "prod" ? 20 : 2
  replica_count  = min(var.requested_replicas, local.max_replicas)
}

resource "kubernetes_deployment" "app" {
  metadata {
    name = "app"
  }

  spec {
    replicas = local.replica_count

    selector {
      match_labels = {
        app = "app"
      }
    }

    template {
      metadata {
        labels = {
          app = "app"
        }
      }

      spec {
        container {
          name  = "app"
          image = var.app_image
        }
      }
    }
  }
}
```

### Using min with a List

```hcl
variable "subnet_available_ips" {
  type    = list(number)
  default = [254, 14, 100, 56]
}

locals {
  # Find the subnet with the fewest available IPs
  min_available = min(var.subnet_available_ips...)
}

output "smallest_subnet_capacity" {
  value = local.min_available  # Returns 14
}
```

### Combining min and max for Clamping

```hcl
variable "raw_replicas" {
  type    = number
  default = 25
}

locals {
  # Clamp replicas between 2 and 10
  clamped_replicas = max(min(var.raw_replicas, 10), 2)
}

output "clamped_replicas" {
  value = local.clamped_replicas  # Returns 10
}
```

## Step-by-Step Usage

1. Identify the value you want to cap.
2. Call `min(value, upper_limit)` to ensure the result never exceeds the limit.
3. For lists, use `min(list...)` to find the smallest element.
4. Validate in `tofu console`:

```bash
tofu console

> min(5, 10, 3)
3
> min([10, 4, 7]...)
4
```

## min vs max

| Use Case | Function |
|----------|----------|
| Ensure a minimum threshold | `max(value, minimum)` |
| Enforce an upper cap | `min(value, maximum)` |
| Clamp within a range | `max(min(value, upper), lower)` |

## Conclusion

The `min` function in OpenTofu is a clean, readable way to enforce upper bounds on resource sizes and counts. Combined with `max` for lower bounds, you can implement clamping logic that keeps your infrastructure allocations within safe, cost-effective ranges regardless of what dynamic inputs provide.
