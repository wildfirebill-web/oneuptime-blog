# How to Use the transpose Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the transpose function in OpenTofu to swap keys and values in a map of lists for inverting role-to-user mappings and permission structures.

## Introduction

The `transpose` function in OpenTofu swaps a `map(list(string))` so that the inner list values become the outer keys, and the outer keys become list values. This is useful for inverting role-to-user or service-to-tag mappings.

## Syntax

```hcl
transpose(map_of_lists)
```

- Input: `map(list(string))`
- Output: `map(list(string))` with keys and values swapped

## Basic Examples

```hcl
output "transposed" {
  value = transpose({
    "role-a" = ["user1", "user2"],
    "role-b" = ["user2", "user3"]
  })
  # Returns:
  # {
  #   "user1" = ["role-a"]
  #   "user2" = ["role-a", "role-b"]
  #   "user3" = ["role-b"]
  # }
}
```

## Practical Use Cases

### Inverting Role Assignments

```hcl
variable "role_assignments" {
  type = map(list(string))
  default = {
    "admin"    = ["alice", "bob"]
    "readonly" = ["bob", "carol", "dave"]
    "operator" = ["alice", "carol"]
  }
}

locals {
  # Get roles for each user
  user_roles = transpose(var.role_assignments)
}

output "alice_roles" {
  value = local.user_roles["alice"]  # ["admin", "operator"]
}
```

### Service-to-Team Mapping

```hcl
variable "team_services" {
  type = map(list(string))
  default = {
    platform = ["api-gateway", "auth-service"]
    data     = ["data-pipeline", "analytics"]
    frontend = ["web-app", "mobile-backend"]
  }
}

locals {
  service_team = transpose(var.team_services)
}

output "api_gateway_team" {
  value = local.service_team["api-gateway"]  # ["platform"]
}
```

### Region-to-Service Mapping

```hcl
variable "service_regions" {
  type = map(list(string))
  default = {
    api    = ["us-east-1", "eu-west-1"]
    worker = ["us-east-1", "us-west-2"]
  }
}

locals {
  region_services = transpose(var.service_regions)
}

output "us_east_1_services" {
  value = local.region_services["us-east-1"]  # ["api", "worker"]
}
```

## Step-by-Step Usage

```bash
tofu console

> transpose({"a" = ["x", "y"], "b" = ["y", "z"]})
{
  "x" = ["a"]
  "y" = ["a", "b"]
  "z" = ["b"]
}
```

## Conclusion

The `transpose` function inverts a `map(list(string))` in OpenTofu, making it easy to switch between "role to users" and "user to roles" perspectives. Use it for IAM policy inversion, service-team mapping, and any scenario where you need both directions of a many-to-many relationship.
