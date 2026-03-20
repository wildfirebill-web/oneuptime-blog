# How to Use the trim Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the trim function in OpenTofu to remove specific leading and trailing characters from strings.

## Introduction

The `trim` function in OpenTofu removes specified characters from both the beginning and end of a string. Unlike `trimspace`, which only removes whitespace, `trim` lets you specify exactly which characters to strip.

## Syntax

```hcl
trim(string, char_set)
```

- **string** — the input string to process
- **char_set** — a string containing all characters to remove (not a regex, not a delimiter)
- Removes any of the specified characters from both ends of the string

## Basic Examples

```hcl
output "trim_spaces" {
  value = trim("  hello  ", " ")    # Returns "hello"
}

output "trim_slashes" {
  value = trim("/path/to/dir/", "/")  # Returns "path/to/dir"
}

output "trim_multiple_chars" {
  value = trim("***hello***", "*")   # Returns "hello"
}

output "trim_multiple_char_set" {
  value = trim("---hello===", "-=")  # Returns "hello"
}
```

Note: The `char_set` is a **set** of characters, not a substring. `trim("xxhelloxx", "x")` removes all leading and trailing `x` characters.

## Practical Use Cases

### Cleaning URL Paths

```hcl
variable "api_base_path" {
  type    = string
  default = "/api/v1/"
}

locals {
  # Remove leading and trailing slashes for consistent joining
  clean_path = trim(var.api_base_path, "/")
}

output "api_path" {
  value = "https://example.com/${local.clean_path}/users"
  # Returns "https://example.com/api/v1/users"
}
```

### Normalizing ARN Suffixes

```hcl
variable "role_name_with_suffix" {
  type    = string
  default = "::my-role::"
}

locals {
  clean_role_name = trim(var.role_name_with_suffix, ":")
}

output "clean_role_name" {
  value = local.clean_role_name  # Returns "my-role"
}
```

### Stripping Quote Characters

```hcl
variable "raw_value" {
  type    = string
  default = "\"my-secret-value\""
}

locals {
  unquoted_value = trim(var.raw_value, "\"")
}

output "unquoted" {
  value = local.unquoted_value  # Returns "my-secret-value"
}
```

### Processing Directory Paths

```hcl
variable "base_dir" {
  type    = string
  default = "///usr/local/bin///"
}

locals {
  # Clean up extra slashes from path edges
  clean_dir = "/${trim(var.base_dir, "/")}"
}

output "clean_path" {
  value = local.clean_dir  # Returns "/usr/local/bin"
}
```

## Step-by-Step Usage

1. Identify the characters you want to remove from string edges.
2. List all those characters as a single string (the char_set).
3. Apply `trim(string, char_set)`.
4. Test in `tofu console`:

```bash
tofu console

> trim("/hello/", "/")
"hello"
> trim("###title###", "#")
"title"
> trim("  hello  ", " \t")
"hello"
```

## trim vs trimspace vs trimprefix/trimsuffix

| Function | What it Does |
|----------|-------------|
| `trim(s, chars)` | Remove specified chars from both ends |
| `trimspace(s)` | Remove whitespace from both ends |
| `trimprefix(s, prefix)` | Remove exact prefix string from start |
| `trimsuffix(s, suffix)` | Remove exact suffix string from end |

## Important: char_set is a Character Set, Not a String

```hcl
# This removes all 'a', 'b', 'c' characters from both ends
trim("abcHELLOcba", "abc")  # Returns "HELLO"

# NOT the same as removing the prefix "abc"
# trimprefix("abcHELLO", "abc")  # Returns "HELLO"
```

## Conclusion

The `trim` function gives you fine-grained control over character removal from string edges in OpenTofu. It is ideal for cleaning up paths, stripping delimiters, removing quote characters, and normalizing values from external sources. When you need to remove specific characters (not just whitespace), `trim` is the right tool.
