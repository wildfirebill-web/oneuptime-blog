# How to Use the flatten Function in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the flatten function in OpenTofu to recursively flatten nested lists into a single flat list for use with for_each and resource definitions.

## Introduction

The `flatten` function in OpenTofu takes a list that may contain nested lists (to any depth) and returns a single flat list containing all the leaf elements. It is commonly used to process the output of `for` expressions that return lists of lists.

## Syntax

```hcl
flatten(list)
```

- **list** - a list that may contain nested lists
- Returns a single-level flat list
- Non-list elements are kept as-is

## Basic Examples

```hcl
output "flatten_nested" {
  value = flatten(["a", ["b", "c"], ["d", ["e"]]])
  # Returns ["a", "b", "c", "d", "e"]
}

output "already_flat" {
  value = flatten([1, 2, 3])  # Returns [1, 2, 3]
}
```

## Practical Use Cases

### Flattening for_each Outputs

```hcl
variable "services" {
  type = map(object({
    ports = list(number)
  }))
  default = {
    api    = { ports = [8080, 8443] }
    worker = { ports = [9090] }
    admin  = { ports = [8081, 8444, 8445] }
  }
}

locals {
  # Generate one object per {service, port} combination
  all_port_configs = flatten([
    for svc, cfg in var.services : [
      for port in cfg.ports : {
        service = svc
        port    = port
      }
    ]
  ])
}

output "all_ports" {
  value = local.all_port_configs
}
```

### Creating Security Group Rules from Nested Map

```hcl
variable "sg_rules" {
  type = map(object({
    cidrs = list(string)
    port  = number
  }))
  default = {
    web = { cidrs = ["0.0.0.0/0"], port = 443 }
    api = { cidrs = ["10.0.0.0/8", "172.16.0.0/12"], port = 8080 }
  }
}

locals {
  flat_rules = flatten([
    for name, rule in var.sg_rules : [
      for cidr in rule.cidrs : {
        name = name
        cidr = cidr
        port = rule.port
      }
    ]
  ])
}

resource "aws_security_group_rule" "ingress" {
  for_each = {
    for r in local.flat_rules :
    "${r.name}-${r.cidr}" => r
  }

  type        = "ingress"
  from_port   = each.value.port
  to_port     = each.value.port
  protocol    = "tcp"
  cidr_blocks = [each.value.cidr]

  security_group_id = aws_security_group.app.id
}
```

### Combining Lists from Multiple Objects

```hcl
variable "environments" {
  type = map(list(string))
  default = {
    prod    = ["us-east-1", "us-west-2"]
    staging = ["us-east-1"]
    dev     = ["us-east-1", "eu-west-1"]
  }
}

locals {
  all_regions = distinct(flatten(values(var.environments)))
}

output "all_deployment_regions" {
  value = local.all_regions
}
```

## Step-by-Step Usage

1. Create a `for` expression that returns a list of lists.
2. Wrap it with `flatten()` to get a flat list.
3. Use the flat list with `for_each` (after converting to map with unique keys).
4. Test in `tofu console`:

```bash
tofu console

> flatten([[1, 2], [3, 4], [5]])
[1, 2, 3, 4, 5]
```

## Conclusion

The `flatten` function is essential for handling the list-of-lists output from nested `for` expressions in OpenTofu. It enables the common pattern of generating one resource per item in a cross-product of maps and lists, which is critical for dynamic multi-resource configurations.
