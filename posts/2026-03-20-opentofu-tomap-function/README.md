# How to Use the tomap Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the tomap function in OpenTofu to convert objects to map type for consistent map operations and dynamic attribute access.

## Introduction

The `tomap` function in OpenTofu converts an object value to a map. Objects in HCL have a fixed schema, while maps are dynamic. Converting to a map allows you to use map functions like `keys()`, `values()`, `merge()`, and dynamic lookups.

## Syntax

```hcl
tomap(object)
```

- Converts an object to a map
- All values must be the same type for a typed map
- Returns `map(type)`

## Basic Examples

```hcl
output "object_to_map" {
  value = tomap({
    name = "example"
    env  = "prod"
  })
  # Returns: {env = "prod", name = "example"}
}
```

## Practical Use Cases

### Dynamic Map Operations on Objects

```hcl
variable "config" {
  type = object({
    db_host = string
    db_port = number
    db_name = string
  })
  default = {
    db_host = "localhost"
    db_port = 5432
    db_name = "appdb"
  }
}

locals {
  # Convert to map for dynamic key iteration
  config_map = tomap({
    db_host = var.config.db_host
    db_port = tostring(var.config.db_port)
    db_name = var.config.db_name
  })
}

resource "aws_ssm_parameter" "configs" {
  for_each = local.config_map

  name  = "/app/config/${each.key}"
  type  = "String"
  value = each.value
}
```

### Converting Resource Attributes to Map

```hcl
locals {
  instance_info = tomap({
    id         = aws_instance.app.id
    private_ip = aws_instance.app.private_ip
    public_ip  = aws_instance.app.public_ip
  })
}

output "instance_details" {
  value = local.instance_info
}
```

### Building Tag Maps Dynamically

```hcl
variable "base_name" {
  type    = string
  default = "myapp"
}

locals {
  standard_tags = tomap({
    Name        = var.base_name
    Environment = var.environment
    ManagedBy   = "OpenTofu"
  })
}
```

## Step-by-Step Usage

```bash
tofu console

> tomap({a = "x", b = "y"})
{a = "x", b = "y"}
> keys(tomap({b = 2, a = 1}))
["a", "b"]
```

## Conclusion

The `tomap` function enables treating object values as maps in OpenTofu. This is useful when you want to iterate over object fields, use map functions, or pass an object where a map type is expected.
