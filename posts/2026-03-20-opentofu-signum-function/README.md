# How to Use the signum Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the signum function in OpenTofu to determine the sign of a number and use it in conditional infrastructure logic.

## Introduction

The `signum` function in OpenTofu returns the sign of a number as `-1`, `0`, or `1`. It is useful for detecting the direction of a change, implementing sign-based conditional logic, or normalizing values to their direction without magnitude.

## Syntax

```hcl
signum(number)
```

- Returns `-1` if the number is negative
- Returns `0` if the number is zero
- Returns `1` if the number is positive

## Basic Examples

```hcl
output "positive_sign" {
  value = signum(42)    # Returns 1
}

output "negative_sign" {
  value = signum(-15)   # Returns -1
}

output "zero_sign" {
  value = signum(0)     # Returns 0
}

output "float_sign" {
  value = signum(-0.01) # Returns -1
}
```

## Practical Use Cases

### Detecting Direction of Resource Change

```hcl
variable "current_replica_count" {
  type    = number
  default = 5
}

variable "target_replica_count" {
  type    = number
  default = 3
}

locals {
  delta     = var.target_replica_count - var.current_replica_count
  direction = signum(local.delta)
  # direction == -1 means scale down
  # direction ==  0 means no change
  # direction ==  1 means scale up
}

output "scaling_direction" {
  value = local.direction == 1 ? "scale-up" : (local.direction == -1 ? "scale-down" : "no-change")
}
```

### Normalizing Priority Values

```hcl
variable "priority_score" {
  type    = number
  default = -7
}

locals {
  # Use only the sign, not the magnitude
  priority_direction = signum(var.priority_score)
  priority_label     = local.priority_direction == 1 ? "high" : (local.priority_direction == -1 ? "low" : "neutral")
}

output "priority_label" {
  value = local.priority_label  # Returns "low"
}
```

### Conditional Resource Tagging

```hcl
variable "cost_delta" {
  type    = number
  description = "Positive means cost increase, negative means savings"
  default = -500
}

locals {
  cost_trend = signum(var.cost_delta)
}

resource "aws_s3_bucket" "reports" {
  bucket = "cost-reports"

  tags = {
    Name      = "cost-reports"
    CostTrend = local.cost_trend == 1 ? "increasing" : (local.cost_trend == -1 ? "decreasing" : "stable")
  }
}
```

## Step-by-Step Usage

1. Compute or receive a numeric value that has directional meaning.
2. Apply `signum()` to reduce it to -1, 0, or 1.
3. Use the result in conditional expressions or tags.
4. Test in `tofu console`:

```bash
tofu console

> signum(100)
1
> signum(-100)
-1
> signum(0)
0
```

## Combining signum with Conditionals

```hcl
locals {
  raw_value = -42
  label = signum(local.raw_value) == -1 ? "below-zero" : "at-or-above-zero"
}
```

## When to Use signum vs Direct Comparison

| Scenario | Preferred Approach |
|----------|-------------------|
| Check if a value is positive | `value > 0` |
| Check if a value is negative | `value < 0` |
| Normalize direction to -1/0/1 | `signum(value)` |
| Use sign as a multiplier | `signum(value) * magnitude` |

## Conclusion

The `signum` function is a niche but useful tool in OpenTofu for sign-based logic. It shines when you need to classify values into three categories (negative, zero, positive) or use the sign as a multiplier in more complex expressions. For simple comparisons, direct boolean operators are usually clearer, but `signum` is the right choice when you explicitly need the numeric sign representation.
