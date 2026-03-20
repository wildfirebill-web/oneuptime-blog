# How to Move Resources with moved Blocks in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Moved, Refactoring, Resources, HCL, Infrastructure as Code, DevOps

Description: Learn how to use moved blocks in OpenTofu to rename or reorganize resources in your configuration without destroying and recreating them.

---

When you rename a resource or move it into a module, OpenTofu sees a new address and a missing old address - it plans to destroy the old resource and create a new one. The `moved` block tells OpenTofu that the resource was just renamed, not destroyed, preventing unnecessary recreation of infrastructure.

---

## The Problem: Renaming Causes Replacement

```hcl
# BEFORE: resource named "old"

resource "aws_instance" "old" {
  ami           = "ami-123"
  instance_type = "t3.micro"
}

# AFTER: resource renamed to "new" - without moved block:
resource "aws_instance" "new" {
  ami           = "ami-123"
  instance_type = "t3.micro"
}

# Plan shows:
# - aws_instance.old will be destroyed
# + aws_instance.new will be created
# Unintended! Your EC2 instance gets destroyed and recreated.
```

---

## The Solution: moved Block

```hcl
# main.tf - add the moved block alongside the renamed resource
resource "aws_instance" "new" {
  ami           = "ami-123"
  instance_type = "t3.micro"
}

# Tell OpenTofu: "old" became "new"
moved {
  from = aws_instance.old
  to   = aws_instance.new
}

# Plan now shows:
# ~ aws_instance.old will be moved to aws_instance.new
# No destruction - just a rename in state
```

---

## Moving into a Module

```hcl
# BEFORE: resource in root module
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# AFTER: resource moved into a networking module
module "networking" {
  source = "./modules/networking"
}

# Add a moved block to tell OpenTofu about the move
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.main
}
```

---

## Moving count Resources

```hcl
# BEFORE: single resource
resource "aws_instance" "web" {
  ami = "ami-123"
}

# AFTER: using count (or vice versa)
resource "aws_instance" "web" {
  count = 1
  ami   = "ami-123"
}

# Moved block for count index
moved {
  from = aws_instance.web
  to   = aws_instance.web[0]
}
```

---

## Moving for_each Resources

```hcl
# BEFORE: single resource
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
}

# AFTER: for_each resource
resource "aws_s3_bucket" "data" {
  for_each = { "main" = "my-data-bucket" }
  bucket   = each.value
}

moved {
  from = aws_s3_bucket.data
  to   = aws_s3_bucket.data["main"]
}
```

---

## Clean Up moved Blocks

After all team members and CI/CD pipelines have applied the moved block, you can remove it. The move is permanent in state.

```bash
# Apply with the moved block
tofu apply

# Later: remove the moved block from config
# The state already has the new address - no move needed on future plans
```

---

## Summary

`moved` blocks allow renaming resources, moving them into modules, and changing from single resources to count/for_each instances without destroying and recreating infrastructure. Add the `moved` block with `from = <old-address>` and `to = <new-address>` alongside the renamed resource. After the team has applied, remove the `moved` block - it's a one-time migration helper.
