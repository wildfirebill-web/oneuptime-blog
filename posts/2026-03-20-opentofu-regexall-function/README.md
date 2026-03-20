# How to Use the regexall Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the regexall function in OpenTofu to find all matches of a pattern in a string for extracting multiple occurrences.

## Introduction

The `regexall` function in OpenTofu applies a regular expression to a string and returns a list of ALL matches (not just the first). Unlike `regex`, it returns an empty list if there are no matches rather than raising an error, making it safe for optional matching.

## Syntax

```hcl
regexall(pattern, string)
```

- Returns a list of all matches
- Each match is a string (no capture groups) or a list/map (with captures)
- Returns `[]` if no matches found

## Basic Examples

```hcl
output "all_numbers" {
  value = regexall("[0-9]+", "abc123def456ghi789")
  # Returns ["123", "456", "789"]
}

output "no_matches" {
  value = regexall("[0-9]+", "no digits here")
  # Returns []
}

output "with_captures" {
  value = regexall("([a-z]+)=([0-9]+)", "a=1 b=2 c=3")
  # Returns [["a", "1"], ["b", "2"], ["c", "3"]]
}
```

## Practical Use Cases

### Extracting All IPs from Text

```hcl
variable "log_text" {
  type    = string
  default = "Connections from 10.0.1.5 and 192.168.0.10 blocked"
}

locals {
  extracted_ips = regexall("[0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+", var.log_text)
}

output "found_ips" {
  value = local.extracted_ips  # ["10.0.1.5", "192.168.0.10"]
}
```

### Counting Occurrences

```hcl
variable "config_text" {
  type    = string
  default = "port=8080\nport=9090\nport=443"
}

locals {
  port_matches = regexall("port=([0-9]+)", var.config_text)
  port_count   = length(local.port_matches)
  ports        = [for m in local.port_matches : tonumber(m[0])]
}

output "ports" {
  value = local.ports  # [8080, 9090, 443]
}
```

### Extracting All Tags from a String

```hcl
variable "tagged_string" {
  type    = string
  default = "[env:prod] [team:platform] [tier:backend]"
}

locals {
  tag_pairs = regexall("\\[([a-z]+):([a-z]+)\\]", var.tagged_string)
  tags_map  = { for pair in local.tag_pairs : pair[0] => pair[1] }
}

output "extracted_tags" {
  value = local.tags_map
  # {env = "prod", team = "platform", tier = "backend"}
}
```

### Safe Pattern Matching (No Error on Miss)

```hcl
variable "maybe_has_version" {
  type    = string
  default = "no-version-here"
}

locals {
  version_matches = regexall("v[0-9]+\\.[0-9]+", var.maybe_has_version)
  has_version     = length(local.version_matches) > 0
  version         = local.has_version ? local.version_matches[0] : "unknown"
}
```

## Step-by-Step Usage

```bash
tofu console

> regexall("[0-9]+", "a1b2c3")
["1", "2", "3"]
> regexall("foo", "bar baz")
[]
> length(regexall("[a-z]+", "hello world"))
2
```

## regexall vs regex

| Function | No match behavior | Returns |
|----------|------------------|---------|
| `regex(p, s)` | Error | First match |
| `regexall(p, s)` | Empty list `[]` | All matches |

Use `regexall` when no match is acceptable; use `regex` when a match is required.

## Conclusion

The `regexall` function is the safe, all-matches variant of `regex` in OpenTofu. Use it when you need to extract all occurrences of a pattern, count matches, or safely test for optional patterns without causing errors when no match is found.
