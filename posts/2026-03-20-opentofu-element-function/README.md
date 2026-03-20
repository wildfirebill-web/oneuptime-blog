# How to Use the element Function in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the element function in OpenTofu to safely access list elements by index with wrapping behavior for round-robin resource distribution.

## Introduction

The `element` function in OpenTofu retrieves an element from a list by index, with wrapping behavior: if the index exceeds the list length, it wraps around. This is particularly useful for distributing resources across a fixed set of options (like availability zones) using a modulo-like approach.

## Syntax

```hcl
element(list, index)
```

- **list** - the list to access
- **index** - the index (0-based); wraps using modulo if >= length
- Returns the element at that position

## Basic Examples

```hcl
variable "azs" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

output "first_az" {
  value = element(var.azs, 0)   # Returns "us-east-1a"
}

output "wrapped_index" {
  value = element(var.azs, 4)   # Returns "us-east-1b" (4 % 3 = 1)
}
```

## Practical Use Cases

### Round-Robin AZ Distribution

```hcl
variable "instance_count" {
  type    = number
  default = 7
}

variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_instance" "app" {
  count             = var.instance_count
  ami               = data.aws_ami.ubuntu.id
  instance_type     = "t3.medium"
  # Distribute instances round-robin across AZs
  availability_zone = element(var.availability_zones, count.index)

  tags = {
    Name = "app-${count.index + 1}"
    AZ   = element(var.availability_zones, count.index)
  }
}
```

### Cycling Through Instance Types

```hcl
variable "worker_count" {
  type    = number
  default = 6
}

locals {
  instance_types = ["t3.small", "t3.medium", "t3.large"]
}

resource "aws_instance" "workers" {
  count         = var.worker_count
  ami           = data.aws_ami.ubuntu.id
  # Cycle through instance types
  instance_type = element(local.instance_types, count.index)

  tags = {
    Name = "worker-${count.index + 1}"
  }
}
```

### Subnet Distribution

```hcl
variable "subnet_ids" {
  type    = list(string)
  default = ["subnet-1", "subnet-2", "subnet-3"]
}

resource "aws_instance" "nodes" {
  count     = 9
  ami       = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"
  subnet_id = element(var.subnet_ids, count.index)

  tags = {
    Name = "node-${count.index + 1}"
  }
}
```

## Step-by-Step Usage

1. Identify the list and the index.
2. Use `element(list, index)` - safe even for out-of-bounds indices due to wrapping.
3. Test in `tofu console`:

```bash
tofu console

> element(["a", "b", "c"], 0)
"a"
> element(["a", "b", "c"], 5)
"c"
```

## element vs Direct Indexing

| Approach | Out-of-bounds behavior |
|----------|----------------------|
| `list[index]` | Error if index >= length |
| `element(list, index)` | Wraps using modulo |

## Conclusion

The `element` function's wrapping behavior makes it ideal for round-robin distribution of resources across availability zones, subnets, and instance types. Use it with `count.index` to automatically cycle through options without bounds checking.
