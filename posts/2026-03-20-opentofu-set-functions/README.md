---
title: "How to Use Set Functions in OpenTofu"
author: nawazdhandala
tags: opentofu, terraform, iac, functions, collections
description: "Learn how to use setintersection, setproduct, setsubtract, and setunion functions in OpenTofu for set operations."
---

# How to Use Set Functions in OpenTofu

OpenTofu provides four set operation functions — `setintersection`, `setproduct`, `setsubtract`, and `setunion`. These operate on sets (or sets created from lists) and are useful for comparing collections, finding differences, and generating combinations.

## setunion()

Returns the union of all provided sets (all unique elements from all sets):

```hcl
> setunion(["a", "b"], ["b", "c"], ["c", "d"])
toset(["a", "b", "c", "d"])
```

```hcl
variable "primary_cidrs" {
  type    = set(string)
  default = ["10.0.0.0/8", "172.16.0.0/12"]
}

variable "additional_cidrs" {
  type    = set(string)
  default = ["192.168.0.0/16", "10.0.0.0/8"]
}

locals {
  all_cidrs = setunion(var.primary_cidrs, var.additional_cidrs)
  # toset(["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"])
}
```

## setintersection()

Returns elements that appear in ALL provided sets:

```hcl
> setintersection(["a", "b", "c"], ["b", "c", "d"])
toset(["b", "c"])
```

```hcl
variable "team_a_access" {
  type    = set(string)
  default = ["s3:GetObject", "s3:PutObject", "ec2:DescribeInstances"]
}

variable "team_b_access" {
  type    = set(string)
  default = ["s3:GetObject", "s3:DeleteObject", "ec2:DescribeInstances"]
}

locals {
  # Permissions both teams share
  shared_access = setintersection(var.team_a_access, var.team_b_access)
  # toset(["ec2:DescribeInstances", "s3:GetObject"])
}
```

## setsubtract()

Returns elements in the first set that are NOT in the second set:

```hcl
> setsubtract(["a", "b", "c"], ["b", "c"])
toset(["a"])
```

```hcl
variable "all_regions" {
  type    = set(string)
  default = ["us-east-1", "us-west-2", "eu-west-1", "ap-southeast-1"]
}

variable "excluded_regions" {
  type    = set(string)
  default = ["ap-southeast-1"]
}

locals {
  active_regions = setsubtract(var.all_regions, var.excluded_regions)
  # toset(["eu-west-1", "us-east-1", "us-west-2"])
}

resource "aws_s3_bucket" "regional" {
  for_each = local.active_regions
  bucket   = "myapp-${each.key}"
}
```

## setproduct()

Returns the Cartesian product of sets (all possible combinations):

```hcl
> setproduct(["dev", "prod"], ["us-east-1", "eu-west-1"])
[
  ["dev", "us-east-1"],
  ["dev", "eu-west-1"],
  ["prod", "us-east-1"],
  ["prod", "eu-west-1"],
]
```

```hcl
locals {
  environments = ["dev", "staging", "prod"]
  services     = ["api", "worker", "frontend"]
  
  # Create all environment-service combinations
  combinations = setproduct(local.environments, local.services)
  # [["dev","api"], ["dev","worker"], ["dev","frontend"],
  #  ["staging","api"], ..., ["prod","frontend"]]
  
  # Convert to map for for_each
  deployments = {
    for combo in local.combinations :
    "${combo[0]}-${combo[1]}" => {
      environment = combo[0]
      service     = combo[1]
    }
  }
}

resource "aws_ecs_service" "deployments" {
  for_each = local.deployments
  
  name            = each.key
  cluster         = aws_ecs_cluster.main.id
  task_definition = "${each.value.service}:${each.value.environment}"
}
```

## Conclusion

Set functions enable powerful collection operations in OpenTofu. Use `setunion()` to combine sets of CIDRs or permissions, `setintersection()` to find shared elements, `setsubtract()` to exclude elements, and `setproduct()` to generate all combinations of multiple sets. These functions are particularly useful for managing IAM permissions, network CIDR blocks, and multi-environment/multi-region deployments.
