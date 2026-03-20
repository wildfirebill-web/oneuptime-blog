# How to Use the matchkeys Function in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the matchkeys function in OpenTofu to filter one list based on matching values in a second parallel list.

## Introduction

The `matchkeys` function in OpenTofu filters elements from one list based on whether corresponding elements in another list match a set of search keys. It enables parallel-list filtering without needing complex `for` expressions.

## Syntax

```hcl
matchkeys(valueslist, keyslist, searchset)
```

- **valueslist** - the list of values to filter
- **keyslist** - the parallel list of keys to match against
- **searchset** - the set of keys to match
- Returns a list of values from `valueslist` where the corresponding `keyslist` element is in `searchset`

## Basic Examples

```hcl
output "matching_values" {
  value = matchkeys(
    ["a", "b", "c", "d"],  # values
    ["w", "x", "y", "z"],  # keys
    ["x", "z"]             # search for these keys
  )
  # Returns ["b", "d"]
}
```

## Practical Use Cases

### Getting Subnet IDs for Specific AZs

```hcl
variable "subnet_ids" {
  type    = list(string)
  default = ["subnet-1", "subnet-2", "subnet-3"]
}

variable "subnet_azs" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "target_azs" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1c"]
}

locals {
  # Get subnet IDs only for the target AZs
  target_subnet_ids = matchkeys(var.subnet_ids, var.subnet_azs, var.target_azs)
}

output "target_subnets" {
  value = local.target_subnet_ids  # ["subnet-1", "subnet-3"]
}
```

### Filtering Instance IDs by Tag

```hcl
variable "instance_ids" {
  type    = list(string)
  default = ["i-001", "i-002", "i-003", "i-004"]
}

variable "instance_envs" {
  type    = list(string)
  default = ["prod", "dev", "prod", "staging"]
}

locals {
  prod_instances = matchkeys(var.instance_ids, var.instance_envs, ["prod"])
}

output "prod_instance_ids" {
  value = local.prod_instances  # ["i-001", "i-003"]
}
```

### Matching AMIs to Regions

```hcl
variable "all_regions" {
  type    = list(string)
  default = ["us-east-1", "us-west-2", "eu-west-1", "ap-southeast-1"]
}

variable "all_amis" {
  type    = list(string)
  default = ["ami-001", "ami-002", "ami-003", "ami-004"]
}

variable "deploy_regions" {
  type    = list(string)
  default = ["us-east-1", "eu-west-1"]
}

locals {
  deploy_amis = matchkeys(var.all_amis, var.all_regions, var.deploy_regions)
}

output "deploy_amis" {
  value = local.deploy_amis  # ["ami-001", "ami-003"]
}
```

## Step-by-Step Usage

1. Have two parallel lists (keys and values).
2. Specify the search keys.
3. Call `matchkeys(values, keys, search)`.
4. Test in `tofu console`:

```bash
tofu console

> matchkeys(["a", "b", "c"], [1, 2, 3], [2, 3])
["b", "c"]
```

## Conclusion

The `matchkeys` function is the parallel-list filter in OpenTofu. Use it when you have two lists where position determines the relationship, and you want to select elements from one list based on matching values in the other. It is particularly useful for AZ/subnet mapping and region/AMI correlation.
