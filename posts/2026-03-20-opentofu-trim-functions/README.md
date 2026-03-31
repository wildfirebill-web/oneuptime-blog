# How to Use Trim Functions in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Function, String

Description: Learn how to use trim, trimspace, trimprefix, and trimsuffix functions in OpenTofu to clean up strings.

OpenTofu provides four trim functions for removing characters from strings: `trim`, `trimspace`, `trimprefix`, and `trimsuffix`. These are useful for cleaning user input, normalizing paths, and removing known prefixes/suffixes.

## trimspace()

Removes all leading and trailing whitespace (spaces, tabs, newlines):

```hcl
> trimspace("  hello world  ")
"hello world"

> trimspace("\t\nhello\n\t")
"hello"
```

```hcl
variable "environment" {
  type = string
}

locals {
  env = trimspace(var.environment)
  # Prevents issues if user adds spaces
}
```

## trim()

Removes specific characters from both ends of a string:

```hcl
> trim("**hello**", "*")
"hello"

> trim("/path/to/dir/", "/")
"path/to/dir"

> trim("xxhelloxx", "x")
"hello"
```

```hcl
locals {
  clean_path = trim(var.s3_prefix, "/")
  # Removes leading/trailing slashes
  
  full_key = "${clean_path}/config.json"
}
```

## trimprefix()

Removes a specific prefix from the start of a string (only once):

```hcl
> trimprefix("https://example.com", "https://")
"example.com"

> trimprefix("arn:aws:iam::123:role/myrole", "arn:aws:iam::123:role/")
"myrole"
```

```hcl
variable "subnet_ids" {
  type    = list(string)
  default = ["subnet-abc123", "subnet-def456"]
}

locals {
  # Extract just the IDs without the "subnet-" prefix
  short_ids = [for id in var.subnet_ids : trimprefix(id, "subnet-")]
  # ["abc123", "def456"]
}
```

## trimsuffix()

Removes a specific suffix from the end of a string:

```hcl
> trimsuffix("hello.tf", ".tf")
"hello"

> trimsuffix("my-bucket-prod", "-prod")
"my-bucket"
```

```hcl
variable "domain" {
  default = "api.example.com."
}

locals {
  # Remove trailing dot from DNS name
  clean_domain = trimsuffix(var.domain, ".")
  # "api.example.com"
}
```

## Practical Example

```hcl
variable "resource_prefix" {
  description = "Prefix for resource names (with or without trailing dash)"
  type        = string
  default     = "myapp-"
}

locals {
  # Normalize: ensure exactly one trailing dash
  prefix = "${trimspace(trimsuffix(var.resource_prefix, "-"))}-"
}

resource "aws_s3_bucket" "data" {
  bucket = "${local.prefix}data"
}

resource "aws_s3_bucket" "logs" {
  bucket = "${local.prefix}logs"
}
```

## Conclusion

The trim functions are your tools for defensive string handling. Use `trimspace()` to clean user-provided variables, `trim()` for removing specific characters from both ends, `trimprefix()` to strip known prefixes, and `trimsuffix()` to remove known suffixes. Together, they help ensure your string inputs are normalized before use in resource configurations.
