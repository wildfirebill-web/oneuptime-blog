# How to Use the trimprefix and trimsuffix Functions in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the trimprefix and trimsuffix functions in OpenTofu to remove exact leading and trailing substrings from strings.

## Introduction

The `trimprefix` and `trimsuffix` functions in OpenTofu remove a specified substring from the beginning or end of a string respectively. Unlike `trim` (which removes characters), these functions remove exact string matches — making them ideal for stripping known prefixes and suffixes from resource names, ARNs, and paths.

## Syntax

```hcl
trimprefix(string, prefix)
trimsuffix(string, suffix)
```

- If the string starts with (or ends with) the given prefix (or suffix), it is removed.
- If it does not match, the original string is returned unchanged.
- Only one occurrence is removed (the prefix or suffix).

## Basic Examples

```hcl
output "remove_prefix" {
  value = trimprefix("hello-world", "hello-")  # Returns "world"
}

output "remove_suffix" {
  value = trimsuffix("hello-world", "-world")  # Returns "hello"
}

output "no_match_prefix" {
  value = trimprefix("hello-world", "world-")  # Returns "hello-world" (unchanged)
}

output "no_match_suffix" {
  value = trimsuffix("hello-world", "hello")  # Returns "hello-world" (unchanged)
}
```

## Practical Use Cases

### Extracting Resource Names from ARNs

```hcl
variable "role_arn" {
  type    = string
  default = "arn:aws:iam::123456789012:role/my-app-role"
}

locals {
  # Extract just the role name from the ARN
  role_name = trimprefix(var.role_arn, "arn:aws:iam::123456789012:role/")
}

output "role_name" {
  value = local.role_name  # Returns "my-app-role"
}
```

### Removing Environment Prefixes from Names

```hcl
variable "resource_names" {
  type = list(string)
  default = [
    "prod-api-service",
    "prod-worker-service",
    "prod-frontend-service"
  ]
}

locals {
  # Strip the "prod-" prefix to get base service names
  base_names = [
    for name in var.resource_names :
    trimprefix(name, "prod-")
  ]
}

output "base_names" {
  value = local.base_names
  # ["api-service", "worker-service", "frontend-service"]
}
```

### Removing File Extensions

```hcl
variable "config_file" {
  type    = string
  default = "app-config.json"
}

locals {
  # Remove .json extension to get base config name
  config_name = trimsuffix(var.config_file, ".json")
}

output "config_name" {
  value = local.config_name  # Returns "app-config"
}
```

### Normalizing URL Schemes

```hcl
variable "endpoint" {
  type    = string
  default = "https://api.example.com"
}

locals {
  # Remove the https:// prefix to get just the hostname
  hostname = trimprefix(trimprefix(var.endpoint, "https://"), "http://")
}

output "hostname" {
  value = local.hostname  # Returns "api.example.com"
}
```

### Stripping Version Suffixes

```hcl
variable "image_tags" {
  type    = list(string)
  default = ["app-1.2.3", "worker-1.2.3", "proxy-1.2.3"]
}

locals {
  # Remove version suffix to get service names
  service_names = [
    for tag in var.image_tags :
    trimsuffix(tag, "-1.2.3")
  ]
}

output "service_names" {
  value = local.service_names  # ["app", "worker", "proxy"]
}
```

## Step-by-Step Usage

1. Identify the exact prefix or suffix to remove.
2. Call `trimprefix(string, prefix)` or `trimsuffix(string, suffix)`.
3. Use the result in further processing or resource definitions.
4. Test in `tofu console`:

```bash
tofu console

> trimprefix("env-production", "env-")
"production"
> trimsuffix("app-service-v2", "-v2")
"app-service"
> trimprefix("hello", "world")  # No match
"hello"
```

## Combining trimprefix and trimsuffix

```hcl
locals {
  wrapped = "[my-value]"
  # Remove both brackets
  unwrapped = trimsuffix(trimprefix(local.wrapped, "["), "]")
  # Returns: "my-value"
}
```

## Key Differences from trim

| Function | Behavior |
|----------|----------|
| `trim(s, chars)` | Removes individual characters from char set |
| `trimprefix(s, prefix)` | Removes exact substring from start |
| `trimsuffix(s, suffix)` | Removes exact substring from end |

`trim("//hello//", "/")` removes all slashes from both ends.
`trimprefix("//hello//", "//")` removes only the leading `//`.

## Conclusion

The `trimprefix` and `trimsuffix` functions provide precise string manipulation in OpenTofu for removing known prefixes and suffixes. They are particularly useful for extracting resource names from ARNs, removing environment tags from identifiers, and normalizing names across your infrastructure codebase.
