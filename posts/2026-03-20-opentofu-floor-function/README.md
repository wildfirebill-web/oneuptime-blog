# How to Use the floor Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the floor function in OpenTofu to round numbers down to the nearest integer for conservative resource allocation.

## Introduction

The `floor` function in OpenTofu returns the largest integer less than or equal to a given number — in other words, it rounds down. This is useful when you want conservative resource allocation, such as computing how many batches fit within a limit, or converting a float to a whole number without exceeding a budget.

## Syntax

```hcl
floor(number)
```

- **number** — any numeric value (integer or float)
- Returns the largest integer less than or equal to the input.

## Basic Examples

```hcl
output "floor_positive" {
  value = floor(1.9)   # Returns 1
}

output "floor_negative" {
  value = floor(-1.1)  # Returns -2 (rounds away from zero)
}

output "floor_whole" {
  value = floor(4.0)   # Returns 4
}
```

## Practical Use Cases

### Budget-Constrained Instance Count

When you have a fixed budget, use `floor` to ensure you never exceed it.

```hcl
variable "monthly_budget_usd" {
  type    = number
  default = 500
}

variable "cost_per_instance_usd" {
  type    = number
  default = 73.6
}

locals {
  # Maximum instances within budget, rounded down
  max_instances = floor(var.monthly_budget_usd / var.cost_per_instance_usd)
}

output "affordable_instance_count" {
  value = local.max_instances  # Returns 6
}
```

### Splitting Data into Even Batches

```hcl
variable "record_count" {
  type    = number
  default = 1050
}

variable "batch_size" {
  type    = number
  default = 100
}

locals {
  # Full batches only — partial batch handled separately
  full_batch_count = floor(var.record_count / var.batch_size)
  remainder        = var.record_count - (local.full_batch_count * var.batch_size)
}

output "full_batches" {
  value = local.full_batch_count  # Returns 10
}

output "remaining_records" {
  value = local.remainder  # Returns 50
}
```

### Converting Float Metrics to Integer Tags

```hcl
variable "cpu_utilization_pct" {
  type    = number
  default = 67.8
}

locals {
  # Store integer CPU band in resource tags
  cpu_band = floor(var.cpu_utilization_pct / 10) * 10
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"

  tags = {
    Name    = "app-server"
    CpuBand = "${local.cpu_band}-${local.cpu_band + 10}"
  }
}
```

## Step-by-Step Usage

1. Identify a fractional number that needs to be rounded down.
2. Apply `floor()` around the expression.
3. Verify with `tofu console`:

```bash
tofu console

> floor(3.9)
3
> floor(3.0)
3
> floor(-3.1)
-4
```

## Difference Between floor, ceil, and round

OpenTofu does not have a built-in `round` function. To achieve rounding to the nearest integer, combine `floor` and addition:

```hcl
locals {
  value       = 2.6
  rounded     = floor(local.value + 0.5)  # Returns 3 (nearest integer)
}
```

## Common Pitfalls

- Applying `floor` to already-integer values is harmless but unnecessary.
- On negative numbers, `floor(-1.1)` returns `-2`, not `-1`. Use `ceil` for negative numbers if you want to round toward zero.

## Conclusion

The `floor` function is the right tool in OpenTofu when you need to round down fractional numbers for conservative resource allocation or batch processing calculations. Whether you are computing instance counts within a budget or splitting workloads into batches, `floor` ensures your infrastructure never accidentally over-provisions or exceeds limits.
