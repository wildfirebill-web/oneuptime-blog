# How to Use the formatlist Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the formatlist function in OpenTofu to apply format strings to lists of values, generating formatted lists of strings in bulk.

## Introduction

The `formatlist` function in OpenTofu applies a format string to each element of one or more lists, returning a new list of formatted strings. It is the list-oriented counterpart to `format` and is ideal for bulk string generation from collections.

## Syntax

```hcl
formatlist(spec, values...)
```

- **spec** — a printf-style format string
- **values** — one or more lists (or scalars) providing values for the placeholders
- Returns a list of formatted strings
- When multiple list arguments are provided, they are zipped together

## Basic Examples

```hcl
output "simple_list_format" {
  value = formatlist("Hello, %s!", ["Alice", "Bob", "Carol"])
  # Returns ["Hello, Alice!", "Hello, Bob!", "Hello, Carol!"]
}

output "numbered_list" {
  value = formatlist("item-%02d", [1, 2, 3, 10])
  # Returns ["item-01", "item-02", "item-03", "item-10"]
}
```

## Practical Use Cases

### Generating Resource Names for a Count

```hcl
variable "az_suffixes" {
  type    = list(string)
  default = ["a", "b", "c"]
}

variable "region" {
  type    = string
  default = "us-east-1"
}

locals {
  # Generate AZ names like "us-east-1a", "us-east-1b", "us-east-1c"
  availability_zones = formatlist("${var.region}%s", var.az_suffixes)
}

output "availability_zones" {
  value = local.availability_zones
  # ["us-east-1a", "us-east-1b", "us-east-1c"]
}
```

### Generating ARN Lists

```hcl
variable "bucket_names" {
  type    = list(string)
  default = ["app-data", "app-logs", "app-backups"]
}

variable "account_id" {
  type    = string
  default = "123456789012"
}

locals {
  bucket_arns = formatlist("arn:aws:s3:::%s", var.bucket_names)
  object_arns = formatlist("arn:aws:s3:::%s/*", var.bucket_names)
}

resource "aws_iam_policy" "s3_access" {
  name = "app-s3-access"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = concat(local.bucket_arns, local.object_arns)
    }]
  })
}
```

### Creating Numbered Instance Names

```hcl
variable "server_count" {
  type    = number
  default = 5
}

locals {
  instance_names = formatlist("web-server-%03d", range(1, var.server_count + 1))
}

output "server_names" {
  value = local.instance_names
  # ["web-server-001", "web-server-002", "web-server-003", "web-server-004", "web-server-005"]
}
```

### Multi-List Zipping with formatlist

```hcl
variable "services" {
  type    = list(string)
  default = ["api", "worker", "scheduler"]
}

variable "ports" {
  type    = list(number)
  default = [8080, 8081, 8082]
}

locals {
  # Zip two lists into formatted strings
  service_endpoints = formatlist("%s:%d", var.services, var.ports)
}

output "service_endpoints" {
  value = local.service_endpoints
  # ["api:8080", "worker:8081", "scheduler:8082"]
}
```

### Generating CIDR Subnet Names

```hcl
variable "subnet_indexes" {
  type    = list(number)
  default = [0, 1, 2, 3]
}

locals {
  subnet_cidrs = formatlist("10.0.%d.0/24", var.subnet_indexes)
}

output "subnet_cidrs" {
  value = local.subnet_cidrs
  # ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}
```

## Step-by-Step Usage

1. Write the format string with `%` placeholders.
2. Pass list arguments for each placeholder.
3. Use the resulting list in resource `for_each`, `count`, or other list-accepting arguments.
4. Test in `tofu console`:

```bash
tofu console

> formatlist("node-%02d", [1, 2, 3])
["node-01", "node-02", "node-03"]
> formatlist("%s.example.com", ["api", "app", "admin"])
["api.example.com", "app.example.com", "admin.example.com"]
```

## Scalar Values in formatlist

If a non-list value is passed, it is repeated for each element of the list:

```hcl
locals {
  env     = "prod"
  services = ["api", "worker", "db"]
  tagged   = formatlist("%s-%s", local.env, local.services)
  # ["prod-api", "prod-worker", "prod-db"]
}
```

## Conclusion

The `formatlist` function is a powerful bulk string generation tool in OpenTofu. It eliminates the need for verbose `for` expressions when you simply need to apply a consistent format string to each element of a list. Use it for generating ARN lists, resource name series, endpoint strings, and any other case where you need the same format applied to an entire collection.
