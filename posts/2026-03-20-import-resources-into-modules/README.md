# How to Import Resources into Modules in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Module

Description: Learn how to import existing cloud resources into OpenTofu module resources using both import blocks and the CLI import command.

## Introduction

When you have existing resources that should be managed within an OpenTofu module, you need to import them using the module's resource address. The module path is included in the import target, ensuring the resource is managed within the correct module context.

## Module Resource Addresses

Resources within modules have addresses like `module.<name>.<resource_type>.<resource_name>`:

```text
module.networking.aws_vpc.main
module.compute.aws_instance.web[0]
module.database.aws_db_instance.primary
```

## Method 1: Import Blocks into Modules

```hcl
# imports.tf

import {
  to = module.networking.aws_vpc.main
  id = "vpc-0a1b2c3d4e5f6789"
}

import {
  to = module.compute.aws_instance.web
  id = "i-0123456789abcdef0"
}
```

The module configuration must define these resources:

```hcl
# main.tf
module "networking" {
  source = "./modules/networking"
  cidr   = "10.0.0.0/16"
}

module "compute" {
  source    = "./modules/compute"
  vpc_id    = module.networking.vpc_id
  subnet_id = module.networking.private_subnet_ids[0]
}
```

```hcl
# modules/networking/main.tf
resource "aws_vpc" "main" {
  cidr_block = var.cidr
  # ...
}
```

## Method 2: CLI Import into Modules

```bash
# Import using the full module address
tofu import 'module.networking.aws_vpc.main' vpc-0a1b2c3d4e5f6789

# Import into indexed module
tofu import 'module.compute[0].aws_instance.web' i-0123456789abcdef0

# Import into for_each module
tofu import 'module.servers["web"].aws_instance.main' i-0123456789abcdef0
```

## Importing into Nested Modules

For deeply nested modules:

```hcl
# Root → networking module → vpc submodule
import {
  to = module.networking.module.vpc.aws_vpc.main
  id = "vpc-0a1b2c3d4e5f6789"
}
```

```bash
# CLI equivalent
tofu import 'module.networking.module.vpc.aws_vpc.main' vpc-0a1b2c3d4e5f6789
```

## Importing into Count-Based Modules

```hcl
module "web_servers" {
  source = "./modules/server"
  count  = 3
}
```

```hcl
import {
  to = module.web_servers[0].aws_instance.main
  id = "i-0000000000000000"
}

import {
  to = module.web_servers[1].aws_instance.main
  id = "i-1111111111111111"
}

import {
  to = module.web_servers[2].aws_instance.main
  id = "i-2222222222222222"
}
```

## Importing into for_each Modules

```hcl
module "environments" {
  for_each = toset(["dev", "staging", "prod"])
  source   = "./modules/environment"
  name     = each.value
}
```

```hcl
import {
  to = module.environments["prod"].aws_vpc.main
  id = "vpc-prod-0a1b2c3d"
}

import {
  to = module.environments["staging"].aws_vpc.main
  id = "vpc-stg-0e1f2a3b"
}
```

## Batch Import with for_each Import Blocks

```hcl
locals {
  existing_vpcs = {
    "dev"     = "vpc-dev-0a1b2c3d"
    "staging" = "vpc-stg-0e1f2a3b"
    "prod"    = "vpc-prod-0g1h2i3j"
  }
}

import {
  for_each = local.existing_vpcs
  to       = module.environments[each.key].aws_vpc.main
  id       = each.value
}
```

## Reconciling Module Configuration

After importing, verify the plan shows no changes:

```bash
tofu plan

# Ideal output:
# No changes. Infrastructure is up-to-date.

# If changes appear, update the module configuration to match reality
```

## Conclusion

Importing resources into modules uses the full module path as the resource address. Import blocks are preferred for their reviewability and for_each support. After importing, always verify that `tofu plan` shows no changes, and update your module configuration to accurately reflect the imported resource's attributes. Remove import blocks after successful import.
