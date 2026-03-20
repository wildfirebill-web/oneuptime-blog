# How to Use the lookup Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the lookup function in OpenTofu to retrieve map values with a default fallback for safe, optional configuration lookups.

## Introduction

The `lookup` function in OpenTofu retrieves the value of a key from a map, returning a default value if the key does not exist. Unlike direct map indexing (`map[key]`), `lookup` is safe for optional keys and makes defaults explicit.

## Syntax

```hcl
lookup(map, key, default)
```

- **map** — the map to search
- **key** — the key to look up
- **default** — returned if the key is not in the map
- Note: the `default` argument is optional in some contexts, but always recommended for safe usage

## Basic Examples

```hcl
variable "instance_types" {
  type = map(string)
  default = {
    dev  = "t3.micro"
    prod = "m5.large"
  }
}

output "dev_type" {
  value = lookup(var.instance_types, "dev", "t3.small")    # Returns "t3.micro"
}

output "staging_type" {
  value = lookup(var.instance_types, "staging", "t3.small") # Returns "t3.small" (default)
}
```

## Practical Use Cases

### Environment-Based Configuration

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

locals {
  instance_type_map = {
    dev     = "t3.micro"
    staging = "t3.medium"
    prod    = "m5.large"
  }

  # Fall back to t3.small for unknown environments
  instance_type = lookup(local.instance_type_map, var.environment, "t3.small")
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type

  tags = {
    Name        = "app"
    Environment = var.environment
  }
}
```

### Region-Specific AMI Lookup

```hcl
variable "ami_ids" {
  type = map(string)
  default = {
    "us-east-1" = "ami-0abcdef1234567890"
    "us-west-2" = "ami-0bcdef1234567890a"
  }
}

variable "region" {
  type    = string
  default = "us-east-1"
}

locals {
  ami_id = lookup(var.ami_ids, var.region, "ami-default")
}
```

### Optional Feature Flags

```hcl
variable "feature_overrides" {
  type    = map(bool)
  default = {
    enable_waf = true
  }
}

locals {
  enable_waf        = lookup(var.feature_overrides, "enable_waf", false)
  enable_monitoring = lookup(var.feature_overrides, "enable_monitoring", true)
  enable_backups    = lookup(var.feature_overrides, "enable_backups", true)
}
```

## Step-by-Step Usage

1. Define a map of options.
2. Call `lookup(map, key, default)` to safely retrieve a value.
3. Test in `tofu console`:

```bash
tofu console

> lookup({a = 1, b = 2}, "a", 0)
1
> lookup({a = 1, b = 2}, "c", 99)
99
```

## lookup vs Direct Access

| Method | Missing key behavior |
|--------|---------------------|
| `map[key]` | Error |
| `lookup(map, key, default)` | Returns default |
| `try(map[key], default)` | Returns default |

## Conclusion

The `lookup` function provides safe, default-aware map access in OpenTofu. Use it whenever a key may not exist in a map to avoid runtime errors and make default values explicit in your configuration.
