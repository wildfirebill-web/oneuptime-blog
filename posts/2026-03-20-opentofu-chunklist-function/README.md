# How to Use the chunklist Function in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the chunklist function in OpenTofu to split a list into fixed-size chunks for batch processing and paginated resource creation.

## Introduction

The `chunklist` function in OpenTofu splits a list into sub-lists of a specified size. This is useful for batch processing, paginated API calls, and distributing resources across groups.

## Syntax

```hcl
chunklist(list, chunk_size)
```

- Returns a list of lists, each of at most `chunk_size` elements
- The last chunk may be smaller if the list doesn't divide evenly

## Basic Examples

```hcl
output "chunked" {
  value = chunklist(["a", "b", "c", "d", "e"], 2)
  # Returns [["a", "b"], ["c", "d"], ["e"]]
}
```

## Practical Use Cases

### Batch Processing Resources

```hcl
variable "instance_ids" {
  type    = list(string)
  default = ["i-001", "i-002", "i-003", "i-004", "i-005"]
}

locals {
  # Process in batches of 2
  batches = chunklist(var.instance_ids, 2)
}

output "batch_count" {
  value = length(local.batches)  # 3 batches
}
```

### Distributing Tags Across Resources

```hcl
variable "all_subnets" {
  type    = list(string)
  default = ["s1", "s2", "s3", "s4", "s5", "s6"]
}

locals {
  # 2 subnets per AZ
  subnet_groups = chunklist(var.all_subnets, 2)
}

output "first_az_subnets" {
  value = local.subnet_groups[0]  # ["s1", "s2"]
}
```

### Limiting IAM Policy Size

```hcl
variable "resource_arns" {
  type    = list(string)
}

locals {
  # IAM policies have limits - chunk ARNs into groups
  arn_chunks = chunklist(var.resource_arns, 25)
}

resource "aws_iam_policy" "access" {
  count = length(local.arn_chunks)
  name  = "access-policy-${count.index}"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject"]
      Resource = local.arn_chunks[count.index]
    }]
  })
}
```

## Step-by-Step Usage

```bash
tofu console

> chunklist([1, 2, 3, 4, 5], 2)
[[1, 2], [3, 4], [5]]
```

## Conclusion

The `chunklist` function is OpenTofu's list partitioner. Use it for batch processing, managing IAM policy size limits, and distributing resources evenly across groups.
