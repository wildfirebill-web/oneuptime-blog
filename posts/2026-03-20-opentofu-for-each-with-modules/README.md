# How to Use for_each with Modules in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Modules

Description: Learn how to use the for_each meta-argument with modules in OpenTofu to create multiple named module instances from a map or set.

## Introduction

The `for_each` meta-argument lets you create multiple instances of a module using a map or set. Each instance is identified by a stable string key rather than a numeric index. This is the preferred approach over `count` when instances have meaningful names and you may add or remove individual instances over time.

## Syntax

```hcl
module "name" {
  source   = "./path/to/module"
  for_each = MAP_OR_SET
}
```

Use `each.key` and `each.value` to access the current iteration's key and value.

## Basic Example with a Set

Create an S3 bucket for each environment:

```hcl
module "bucket" {
  source   = "./modules/s3-bucket"
  for_each = toset(["dev", "staging", "prod"])

  bucket_name = "myapp-${each.key}"
  environment = each.key
}
```

## Using a Map for Richer Configuration

Maps let you pass different configuration per instance:

```hcl
variable "environments" {
  type = map(object({
    instance_count = number
    instance_type  = string
  }))
  default = {
    dev = {
      instance_count = 1
      instance_type  = "t3.micro"
    }
    prod = {
      instance_count = 3
      instance_type  = "m5.large"
    }
  }
}

module "app_cluster" {
  source   = "./modules/app-cluster"
  for_each = var.environments

  name           = each.key
  instance_count = each.value.instance_count
  instance_type  = each.value.instance_type
}
```

## Accessing Outputs from for_each Modules

Outputs from `for_each` modules are accessed using the instance key:

```hcl
output "cluster_endpoints" {
  # Returns a map of environment name → endpoint
  value = { for k, v in module.app_cluster : k => v.endpoint }
}

output "prod_endpoint" {
  value = module.app_cluster["prod"].endpoint
}
```

## Multi-Region Deployment

```hcl
variable "regions" {
  type = map(object({
    cidr = string
  }))
  default = {
    us-east-1 = { cidr = "10.0.0.0/16" }
    eu-west-1 = { cidr = "10.1.0.0/16" }
    ap-east-1 = { cidr = "10.2.0.0/16" }
  }
}

module "vpc" {
  source   = "./modules/vpc"
  for_each = var.regions

  name   = "vpc-${each.key}"
  region = each.key
  cidr   = each.value.cidr
}
```

## Adding and Removing Instances

A key advantage of `for_each` over `count`: adding or removing an instance only affects that specific instance:

```hcl
# Adding "uat" to the map only creates module.app_cluster["uat"]
# Other instances are unaffected
variable "environments" {
  default = {
    dev     = { ... }
    staging = { ... }
    uat     = { ... }  # New — only this instance is created
    prod    = { ... }
  }
}
```

With `count`, inserting an item in the middle would change all subsequent indexes and trigger unnecessary replacements.

## Important Notes

- `for_each` values must be known at plan time; they cannot depend on resource outputs.
- Use `toset()` when iterating over a list of strings without extra configuration.
- State keys look like `module.name["key"]`, which is stable across map changes.

## Conclusion

`for_each` with modules is the most flexible way to create multiple named module instances. It produces stable state keys that survive additions and removals, and allows different configuration per instance through map values. Use it whenever instances have meaningful identifiers or when you need to manage them independently.
