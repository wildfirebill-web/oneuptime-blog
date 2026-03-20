# How to Use the replace Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the replace function in OpenTofu to substitute substrings and patterns in strings for generating resource names and normalizing values.

## Introduction

The `replace` function in OpenTofu substitutes occurrences of a substring (or regex pattern) in a string with a replacement value. It is one of the most frequently used string functions for generating resource names, slugifying text, and normalizing identifiers.

## Syntax

```hcl
replace(string, search, replacement)
```

- **string** - the input string
- **search** - the substring or regex pattern to find
- **replacement** - the string to substitute
- Returns the modified string with all occurrences replaced
- If **search** is wrapped in `/` (e.g., `/pattern/`), it is treated as a regex

## Basic Examples

```hcl
output "simple_replace" {
  value = replace("hello world", "world", "OpenTofu")  # Returns "hello OpenTofu"
}

output "replace_spaces" {
  value = replace("my service name", " ", "-")  # Returns "my-service-name"
}

output "regex_replace" {
  value = replace("abc123def456", "/[0-9]+/", "NUM")  # Returns "abcNUMdefNUM"
}
```

## Practical Use Cases

### Creating URL-Safe Slugs

```hcl
variable "display_name" {
  type    = string
  default = "My Amazing Service Name"
}

locals {
  slug = lower(replace(var.display_name, " ", "-"))
}

resource "aws_s3_bucket" "service" {
  bucket = "content-${local.slug}"  # "content-my-amazing-service-name"
}
```

### Normalizing Region Names for Resource Naming

```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

locals {
  # Replace hyphens with underscores for use in parameter names
  region_underscore = replace(var.aws_region, "-", "_")
}

resource "aws_ssm_parameter" "region_config" {
  name  = "/config/${local.region_underscore}/endpoint"
  type  = "String"
  value = "https://api.${var.aws_region}.amazonaws.com"
}
```

### Sanitizing User Input for Resource Names

```hcl
variable "project_name" {
  type    = string
  default = "My Project! 2024 (v2)"
}

locals {
  # Remove special characters, replace spaces with hyphens
  safe_name = lower(replace(replace(var.project_name, "/[^a-zA-Z0-9 ]/", ""), " ", "-"))
}

output "safe_name" {
  value = local.safe_name  # Returns "my-project-2024-v2"
}
```

### Replacing Environment Placeholders in Templates

```hcl
variable "config_template" {
  type    = string
  default = "endpoint: https://{{ENV}}.api.example.com\nenv: {{ENV}}"
}

variable "environment" {
  type    = string
  default = "production"
}

locals {
  config = replace(var.config_template, "{{ENV}}", var.environment)
}

output "resolved_config" {
  value = local.config
}
```

### Converting Naming Conventions

```hcl
variable "camel_case_name" {
  type    = string
  default = "myServiceName"
}

locals {
  # Convert camelCase to kebab-case using regex
  kebab_case = lower(replace(var.camel_case_name, "/([A-Z])/", "-$1"))
}

output "kebab_name" {
  value = local.kebab_case  # Returns "my-service-name"
}
```

### Stripping Unwanted Characters

```hcl
variable "account_number" {
  type    = string
  default = "1234-5678-9012"
}

locals {
  # Remove hyphens from account number
  clean_account = replace(var.account_number, "-", "")
}

output "clean_account_number" {
  value = local.clean_account  # Returns "123456789012"
}
```

## Step-by-Step Usage

1. Identify the search pattern (substring or regex).
2. Determine the replacement string.
3. Apply `replace(string, search, replacement)`.
4. Test in `tofu console`:

```bash
tofu console

> replace("hello world", "world", "there")
"hello there"
> replace("a.b.c", ".", "-")
"a-b-c"
> replace("abc123", "/[0-9]/", "X")
"abcXXX"
```

## Regex Support

Wrap the search in `/` to use regex:

```hcl
locals {
  # Replace all digits
  no_digits = replace("abc123def456", "/[0-9]+/", "")

  # Replace multiple spaces with single space
  single_spaced = replace("hello    world", "/[ ]+/", " ")
}
```

## Conclusion

The `replace` function is one of the most versatile string manipulation tools in OpenTofu. From creating URL-safe slugs to normalizing region names, sanitizing inputs, and replacing template placeholders, `replace` handles both literal string substitution and regex-based pattern replacement. Master this function and you'll have powerful control over string generation throughout your infrastructure code.
