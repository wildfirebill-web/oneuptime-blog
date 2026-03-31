# How to Import Resources into Modules in OpenTofu - Opentofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Import, Module

Description: Learn how to import existing infrastructure resources into OpenTofu modules using import blocks and the tofu import command.

## Introduction

When adopting OpenTofu for existing infrastructure, resources often need to be imported into module-managed configurations rather than the root configuration. Import blocks and the CLI import command both support module-qualified addresses, allowing you to import directly into the correct module hierarchy.

## Import Block into a Module

```hcl
# Root module: import.tf

import {
  to = module.networking.aws_vpc.main
  id = "vpc-0abc123456"
}

# Module configuration: modules/networking/main.tf
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

## Import into a Nested Module

```hcl
# Root → networking module → vpc submodule
import {
  to = module.networking.module.vpc.aws_vpc.main
  id = "vpc-0abc123456"
}
```

## CLI Import into a Module

```bash
# tofu import with module-qualified address
tofu import 'module.networking.aws_vpc.main' vpc-0abc123456

# Nested module
tofu import 'module.networking.module.vpc.aws_vpc.main' vpc-0abc123456
```

## Import Multiple Resources into a Module

```hcl
# import.tf
import {
  to = module.networking.aws_vpc.main
  id = "vpc-0abc123456"
}

import {
  to = module.networking.aws_subnet.public["us-east-1a"]
  id = "subnet-0abc111"
}

import {
  to = module.networking.aws_subnet.public["us-east-1b"]
  id = "subnet-0abc222"
}
```

## Module Configuration Must Declare the Resource

The target resource must exist in the module's configuration:

```hcl
# modules/networking/main.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
}

resource "aws_subnet" "public" {
  for_each          = var.azs
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_cidrs[each.key]
  availability_zone = each.key
}
```

## Import into a count Module

```hcl
# Module called with count
module "app" {
  count  = 2
  source = "./modules/app"
}

# Import into the second instance (index 1)
import {
  to = module.app[1].aws_instance.main
  id = "i-0abc123456"
}
```

## Import into a for_each Module

```hcl
# Module called with for_each
module "region" {
  for_each = toset(["us-east-1", "eu-west-1"])
  source   = "./modules/region"
}

# Import into a specific region's module
import {
  to = module.region["us-east-1"].aws_vpc.main
  id = "vpc-0abc123456"
}
```

## Verifying the Import

```bash
# After importing, verify the resource is in state
tofu state list | grep module.networking

# Show the imported resource
tofu state show 'module.networking.aws_vpc.main'

# Run plan to ensure configuration matches imported state
tofu plan
# Should show no changes if configuration is correct
```

## Reconciling Configuration with Imported State

```bash
tofu plan

# If plan shows changes, the configuration differs from the imported resource
# ~ module.networking.aws_vpc.main will be updated in-place
#   ~ enable_dns_hostnames = false -> true  (actual vs config)

# Update the module configuration to match
# Then re-plan until there are no changes
```

## Conclusion

Importing into modules uses the same import block syntax with a module-qualified address (`module.name.resource_type.resource_name`). The module's configuration must declare the resource before the import can succeed. After importing, verify with `tofu state list` and reconcile any attribute differences by updating the module configuration until `tofu plan` shows no changes.
