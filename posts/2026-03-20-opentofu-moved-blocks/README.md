# Using moved Blocks in OpenTofu for Safe Refactoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Module, Refactoring, Moved

Description: Learn how to use moved blocks in OpenTofu to safely rename and reorganize resources without destroying and recreating them.

The `moved` block is one of OpenTofu's most useful features for refactoring infrastructure code. It lets you rename resources, move them between modules, or change from `count` to `for_each` - all without destroying and recreating the actual infrastructure.

## Basic moved Block Syntax

```hcl
moved {
  from = aws_instance.old_name
  to   = aws_instance.new_name
}
```

This tells OpenTofu: "the resource that was previously tracked as `aws_instance.old_name` is now `aws_instance.new_name` - don't recreate it."

## Renaming a Resource

```hcl
# Before refactoring - resource was called "server"

resource "aws_instance" "server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
}

# After refactoring - renamed to "web_server"
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
}

# moved block tells OpenTofu about the rename
moved {
  from = aws_instance.server
  to   = aws_instance.web_server
}
```

## Moving a Resource into a Module

```hcl
# Before: resource in root module
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = var.vpc_id
}

# After: moved inside a module
module "app" {
  source = "./modules/application"
  vpc_id = var.vpc_id
}

# modules/application/main.tf
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = var.vpc_id
}

# moved block in root main.tf
moved {
  from = aws_security_group.app
  to   = module.app.aws_security_group.app
}
```

## Moving a Resource out of a Module

```hcl
# Before: resource was inside a module
# modules/network/main.tf had aws_eip.nat

# After: moved to root module
resource "aws_eip" "nat" {
  domain = "vpc"
}

moved {
  from = module.network.aws_eip.nat
  to   = aws_eip.nat
}
```

## Migrating from count to for_each

This is one of the most powerful uses of `moved`:

```hcl
# Before: using count (fragile - index-based)
resource "aws_iam_user" "developers" {
  count = length(var.developer_names)
  name  = var.developer_names[count.index]
}

# After: using for_each (stable - key-based)
resource "aws_iam_user" "developers" {
  for_each = toset(var.developer_names)
  name     = each.key
}

# moved blocks for each user
moved {
  from = aws_iam_user.developers[0]
  to   = aws_iam_user.developers["alice"]
}

moved {
  from = aws_iam_user.developers[1]
  to   = aws_iam_user.developers["bob"]
}

moved {
  from = aws_iam_user.developers[2]
  to   = aws_iam_user.developers["charlie"]
}
```

## Moving Between Modules

```hcl
# Move a resource from one module to another
moved {
  from = module.old_module.aws_s3_bucket.logs
  to   = module.new_module.aws_s3_bucket.logs
}

# Move entire module to a different module path
moved {
  from = module.old_module
  to   = module.new_module
}
```

## Practical Refactoring: Restructuring a Module

```hcl
# Before: everything in root module
resource "aws_vpc" "main" { ... }
resource "aws_subnet" "public" { ... }
resource "aws_subnet" "private" { ... }
resource "aws_internet_gateway" "main" { ... }

# After: extracted into networking module
module "networking" {
  source     = "./modules/networking"
  vpc_cidr   = "10.0.0.0/16"
  az_count   = 2
}

# moved blocks - all in one file (moved.tf)
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.main
}

moved {
  from = aws_subnet.public
  to   = module.networking.aws_subnet.public
}

moved {
  from = aws_subnet.private
  to   = module.networking.aws_subnet.private
}

moved {
  from = aws_internet_gateway.main
  to   = module.networking.aws_internet_gateway.main
}
```

## Removing moved Blocks After Migration

Once all team members and CI/CD pipelines have applied the refactored configuration, you can safely remove the `moved` blocks:

```bash
# Verify the plan shows no changes
tofu plan

# If clean, remove the moved blocks from your code
# Then verify again
tofu plan
# Should still show: No changes. Infrastructure is up-to-date.
```

## Conclusion

The `moved` block is essential for maintaining infrastructure without disruption during refactoring. Use it when renaming resources, extracting resources into modules, or migrating from `count` to `for_each`. Keep `moved` blocks until everyone has run `tofu apply`, then clean them up to keep your codebase tidy.
