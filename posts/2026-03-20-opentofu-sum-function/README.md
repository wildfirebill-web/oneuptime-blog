# How to Use the sum Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the sum function in OpenTofu to add up all numbers in a list for aggregating resource counts and cost calculations.

## Introduction

The `sum` function in OpenTofu returns the sum of all numbers in a given list. It is especially useful for aggregating resource counts, computing total storage, summing costs, and combining values from dynamic collections in your infrastructure configurations.

## Syntax

```hcl
sum(list)
```

- **list** — a list of numbers
- Returns the arithmetic total of all elements

## Basic Examples

```hcl
output "simple_sum" {
  value = sum([1, 2, 3, 4, 5])  # Returns 15
}

output "float_sum" {
  value = sum([1.5, 2.5, 3.0])  # Returns 7
}

output "single_element" {
  value = sum([42])  # Returns 42
}
```

## Practical Use Cases

### Summing Replica Counts Across Services

```hcl
variable "service_replicas" {
  type = map(number)
  default = {
    api     = 3
    worker  = 5
    frontend = 2
  }
}

locals {
  total_replicas = sum(values(var.service_replicas))
}

output "total_pod_count" {
  value = local.total_replicas  # Returns 10
}
```

### Calculating Total Storage Requirements

```hcl
variable "volume_sizes_gb" {
  type    = list(number)
  default = [20, 50, 100, 200]
}

locals {
  total_storage_gb = sum(var.volume_sizes_gb)
}

output "total_storage_tb" {
  value = local.total_storage_gb / 1024  # Convert GB to TB
}
```

### Budget Aggregation

```hcl
variable "service_costs" {
  type = map(number)
  default = {
    compute   = 450.00
    storage   = 120.00
    network   = 85.50
    database  = 200.00
  }
}

locals {
  total_monthly_cost = sum(values(var.service_costs))
}

output "total_monthly_cost_usd" {
  value = local.total_monthly_cost  # Returns 855.5
}

resource "aws_budgets_budget" "total" {
  name         = "total-monthly-budget"
  budget_type  = "COST"
  limit_amount = tostring(local.total_monthly_cost * 1.2)  # 20% buffer
  limit_unit   = "USD"
  time_unit    = "MONTHLY"
}
```

### Summing Counts from for_each Resources

```hcl
variable "clusters" {
  type = map(object({
    node_count = number
  }))
  default = {
    primary   = { node_count = 5 }
    secondary = { node_count = 3 }
    edge      = { node_count = 2 }
  }
}

locals {
  total_nodes = sum([for cluster in var.clusters : cluster.node_count])
}

output "total_cluster_nodes" {
  value = local.total_nodes  # Returns 10
}
```

## Step-by-Step Usage

1. Gather the numbers you want to total into a list.
2. Call `sum(list)` with that list.
3. Use the result in outputs, locals, or resource arguments.
4. Test in `tofu console`:

```bash
tofu console

> sum([10, 20, 30])
60
> sum([1.5, 2.5])
4
```

## Combining sum with Other Functions

```hcl
locals {
  raw_values   = [1, -2, 3, -4, 5]
  abs_values   = [for v in local.raw_values : abs(v)]
  abs_total    = sum(local.abs_values)  # Returns 15
}
```

## Common Pitfalls

- Passing an empty list `sum([])` will cause an error. Ensure the list has at least one element.
- Mixing number types (integers and floats) is fine — OpenTofu handles coercion automatically.
- Non-numeric values in the list will cause a type error.

## Conclusion

The `sum` function is a simple but powerful aggregation tool in OpenTofu. Use it whenever you need to total a list of numbers — from replica counts to storage sizes to budget calculations. Combined with list comprehensions and `values()`, it enables concise and expressive aggregate calculations over complex data structures.
