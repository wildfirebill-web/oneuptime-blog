# How to Use the setunion Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the setunion function in OpenTofu to combine multiple sets into one deduplicated set for merging permissions and resource collections.

## Introduction

The `setunion` function in OpenTofu returns a set containing all unique elements from all provided sets. It is the set-based alternative to `concat` + `distinct` and is ideal for merging permissions, combining resource lists, and creating comprehensive allow-lists.

## Syntax

```hcl
setunion(sets...)
```

- Returns a set with all unique elements from all inputs
- Duplicates are automatically removed (set semantics)

## Basic Examples

```hcl
output "union" {
  value = setunion(["a", "b"], ["b", "c"], ["c", "d"])
  # Returns toset(["a", "b", "c", "d"])
}
```

## Practical Use Cases

### Combining Permissions from Multiple Roles

```hcl
variable "base_permissions" {
  type    = list(string)
  default = ["s3:GetObject", "s3:ListBucket"]
}

variable "write_permissions" {
  type    = list(string)
  default = ["s3:PutObject", "s3:DeleteObject"]
}

variable "admin_permissions" {
  type    = list(string)
  default = ["s3:GetObject", "s3:*"]
}

locals {
  all_permissions = setunion(
    toset(var.base_permissions),
    toset(var.write_permissions)
  )
}

resource "aws_iam_policy" "combined" {
  name = "combined-s3"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = tolist(local.all_permissions)
      Resource = "*"
    }]
  })
}
```

### Merging Security Group Collections

```hcl
variable "default_sgs" {
  type    = list(string)
  default = ["sg-001", "sg-002"]
}

variable "service_sgs" {
  type    = list(string)
  default = ["sg-002", "sg-003"]
}

locals {
  all_sgs = setunion(toset(var.default_sgs), toset(var.service_sgs))
}

resource "aws_instance" "app" {
  ami             = data.aws_ami.ubuntu.id
  instance_type   = "t3.medium"
  security_groups = tolist(local.all_sgs)  # ["sg-001", "sg-002", "sg-003"]
}
```

### Combining Allowed CIDRs

```hcl
locals {
  office_cidrs    = toset(["203.0.113.0/24"])
  vpn_cidrs       = toset(["10.200.0.0/16"])
  internal_cidrs  = toset(["10.0.0.0/8", "172.16.0.0/12"])

  all_allowed_cidrs = setunion(
    local.office_cidrs,
    local.vpn_cidrs,
    local.internal_cidrs
  )
}
```

## Step-by-Step Usage

```bash
tofu console

> setunion(["a", "b"], ["b", "c"])
toset(["a", "b", "c"])
```

## setunion vs concat + distinct

```hcl
# These are equivalent:
setunion(toset(list1), toset(list2))
distinct(concat(list1, list2))
```

`setunion` is cleaner and semantically clearer when the intent is set union.

## Conclusion

The `setunion` function provides clean, deduplicated combination of multiple collections in OpenTofu. Use it for merging permission sets, combining security group lists, and building comprehensive allow-lists from multiple sources. It is the preferred alternative to `concat` + `distinct` when set semantics are intended.
