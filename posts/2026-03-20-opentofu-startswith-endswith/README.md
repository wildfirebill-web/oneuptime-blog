# How to Use the startswith and endswith Functions in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the startswith and endswith functions in OpenTofu to test string prefixes and suffixes for validation and conditional logic.

## Introduction

The `startswith` and `endswith` functions in OpenTofu test whether a string begins or ends with a specific substring. They return a boolean result and are commonly used in variable validation, conditional resource configuration, and filtering lists of strings.

## Syntax

```hcl
startswith(string, prefix)
endswith(string, suffix)
```

Both return `true` if the string starts/ends with the given prefix/suffix, and `false` otherwise.

## Basic Examples

```hcl
output "starts_with_true" {
  value = startswith("hello-world", "hello")  # Returns true
}

output "starts_with_false" {
  value = startswith("hello-world", "world")  # Returns false
}

output "ends_with_true" {
  value = endswith("app.js", ".js")           # Returns true
}

output "ends_with_false" {
  value = endswith("app.js", ".ts")           # Returns false
}
```

## Practical Use Cases

### Variable Validation

```hcl
variable "s3_bucket_name" {
  type = string

  validation {
    condition     = startswith(var.s3_bucket_name, "mycompany-")
    error_message = "S3 bucket names must start with 'mycompany-' for compliance."
  }
}

variable "docker_image" {
  type = string

  validation {
    condition     = startswith(var.docker_image, "registry.mycompany.com/")
    error_message = "Docker images must be pulled from the internal registry."
  }
}

variable "ssl_certificate_arn" {
  type = string

  validation {
    condition     = startswith(var.ssl_certificate_arn, "arn:aws:acm:")
    error_message = "ssl_certificate_arn must be an ACM certificate ARN."
  }
}
```

### Detecting Instance Type Families

```hcl
variable "instance_type" {
  type    = string
  default = "t3.medium"
}

locals {
  is_t_family     = startswith(var.instance_type, "t")
  is_m_family     = startswith(var.instance_type, "m")
  is_burstable    = startswith(var.instance_type, "t2") || startswith(var.instance_type, "t3")
}

output "is_burstable" {
  value = local.is_burstable
}
```

### Filtering Files by Extension

```hcl
variable "config_files" {
  type = list(string)
  default = [
    "app.yaml",
    "database.json",
    "network.yaml",
    "secrets.json",
    "README.md"
  ]
}

locals {
  yaml_files = [for f in var.config_files : f if endswith(f, ".yaml")]
  json_files = [for f in var.config_files : f if endswith(f, ".json")]
}

output "yaml_configs" {
  value = local.yaml_files  # ["app.yaml", "network.yaml"]
}
```

### Detecting Environment from Resource Names

```hcl
variable "resource_names" {
  type = list(string)
  default = [
    "prod-api",
    "staging-api",
    "dev-api",
    "prod-db",
    "dev-db"
  ]
}

locals {
  prod_resources    = [for r in var.resource_names : r if startswith(r, "prod-")]
  dev_resources     = [for r in var.resource_names : r if startswith(r, "dev-")]
}

output "production_resources" {
  value = local.prod_resources  # ["prod-api", "prod-db"]
}
```

### Validating ARN Format

```hcl
variable "resource_arns" {
  type = list(string)
  default = [
    "arn:aws:iam::123456789012:role/my-role",
    "arn:aws:s3:::my-bucket",
    "invalid-arn-format"
  ]
}

locals {
  valid_arns   = [for arn in var.resource_arns : arn if startswith(arn, "arn:aws:")]
  invalid_arns = [for arn in var.resource_arns : arn if !startswith(arn, "arn:aws:")]
}

output "invalid_arns" {
  value = local.invalid_arns  # ["invalid-arn-format"]
}
```

## Step-by-Step Usage

1. Determine whether you need a prefix (`startswith`) or suffix (`endswith`) check.
2. Call the function with the string and the prefix/suffix.
3. Use the boolean in `if` expressions, `validation` blocks, or ternary operators.
4. Test in `tofu console`:

```bash
tofu console

> startswith("prod-database", "prod-")
true
> endswith("index.html", ".html")
true
> startswith("myapp", "prod-")
false
```

## Case Sensitivity

Both functions are case-sensitive:

```hcl
locals {
  result = startswith(lower("PROD-DB"), "prod-")  # true (after lowercasing)
}
```

## Conclusion

The `startswith` and `endswith` functions are clean, readable tools for prefix and suffix testing in OpenTofu. Use them in variable validation to enforce naming conventions, in list filtering to select resources by pattern, and in conditional expressions to drive resource configuration based on naming schemes.
