# How to Use the one Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the one function in OpenTofu to safely extract a single element from a list that must contain exactly zero or one items.

## Introduction

The `one` function in OpenTofu takes a list that must have zero or one elements and returns either `null` (if empty) or the single element. It is the idiomatic way to handle `count = 0 or 1` resource outputs in OpenTofu.

## Syntax

```hcl
one(list)
```

- If the list is empty: returns `null`
- If the list has exactly one element: returns that element
- If the list has more than one element: raises an error

## Basic Examples

```hcl
output "empty_list" {
  value = one([])      # Returns null
}

output "single_element" {
  value = one(["hi"]) # Returns "hi"
}
```

## Practical Use Cases

### Accessing Optional Count-Based Resources

```hcl
variable "create_bastion" {
  type    = bool
  default = true
}

resource "aws_instance" "bastion" {
  count         = var.create_bastion ? 1 : 0
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name = "bastion"
  }
}

output "bastion_ip" {
  # one() safely handles the 0-or-1 count pattern
  value = one(aws_instance.bastion[*].public_ip)
}
```

### Optional Data Source Result

```hcl
variable "lookup_existing_vpc" {
  type    = bool
  default = false
}

data "aws_vpc" "existing" {
  count = var.lookup_existing_vpc ? 1 : 0

  filter {
    name   = "tag:Name"
    values = ["my-vpc"]
  }
}

locals {
  vpc_id = one(data.aws_vpc.existing[*].id)
}
```

### Conditional Module Output

```hcl
module "cdn" {
  count  = var.enable_cdn ? 1 : 0
  source = "./modules/cdn"
}

output "cdn_domain" {
  value = one(module.cdn[*].domain_name)
  # Returns null if CDN not enabled
}
```

### Using one with for_each

```hcl
resource "aws_eip" "nat" {
  for_each = var.create_nat_gateway ? { "main" = true } : {}
  domain   = "vpc"
}

output "nat_eip" {
  value = one(values(aws_eip.nat)[*].public_ip)
}
```

## Step-by-Step Usage

```bash
tofu console

> one([])
null
> one(["value"])
"value"
> one(["a", "b"])  # Error: list must have 0 or 1 elements
```

## one vs try

```hcl
# Both handle the 0-or-1 pattern:
output "using_one" {
  value = one(aws_instance.optional[*].id)
}

output "using_try" {
  value = try(aws_instance.optional[0].id, null)
}
```

## Conclusion

The `one` function is the idiomatic OpenTofu way to handle resources created with `count = 0 or 1`. Use it with the splat operator `[*]` to safely extract optional resource attributes without index errors.
