# How to Use the substr Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the substr function in OpenTofu to extract substrings by position and length for parsing identifiers and creating abbreviations.

## Introduction

The `substr` function in OpenTofu extracts a portion of a string by specifying a starting offset and a length. It is useful for extracting prefixes, abbreviations, region codes, and other fixed-position data from strings in your infrastructure configurations.

## Syntax

```hcl
substr(string, offset, length)
```

- **string** — the input string
- **offset** — the starting position (0-indexed; negative values count from the end)
- **length** — the number of characters to extract (-1 for all remaining)
- Returns the substring

## Basic Examples

```hcl
output "first_5_chars" {
  value = substr("hello world", 0, 5)   # Returns "hello"
}

output "from_position_6" {
  value = substr("hello world", 6, 5)   # Returns "world"
}

output "last_5_chars" {
  value = substr("hello world", -5, 5)  # Returns "world"
}

output "from_offset_to_end" {
  value = substr("hello world", 6, -1)  # Returns "world"
}
```

## Practical Use Cases

### Extracting Region Abbreviations

```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

locals {
  # Take first 2 characters as region abbreviation
  region_abbrev = substr(var.aws_region, 0, 2)
}

output "region_abbrev" {
  value = local.region_abbrev  # Returns "us"
}
```

### Generating Short Resource Name Prefixes

```hcl
variable "service_name" {
  type    = string
  default = "authentication-service"
}

locals {
  # Use first 6 chars as short identifier
  short_name = substr(var.service_name, 0, 6)
}

resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "logs" {
  bucket = "${local.short_name}-logs-${random_id.suffix.hex}"
  # "authen-logs-a1b2c3d4"
}
```

### Extracting Account ID Portions

```hcl
data "aws_caller_identity" "current" {}

locals {
  account_id    = data.aws_caller_identity.current.account_id
  # First 4 digits of account ID for grouping
  account_group = substr(local.account_id, 0, 4)
}

output "account_group" {
  value = local.account_group
}
```

### Extracting Date Components from ISO Strings

```hcl
variable "iso_date" {
  type    = string
  default = "2026-03-20T14:30:00Z"
}

locals {
  date_only  = substr(var.iso_date, 0, 10)   # "2026-03-20"
  year       = substr(var.iso_date, 0, 4)    # "2026"
  month      = substr(var.iso_date, 5, 2)    # "03"
  day        = substr(var.iso_date, 8, 2)    # "20"
}

output "year" {
  value = local.year
}
```

### Extracting Hex Color Components

```hcl
variable "hex_color" {
  type    = string
  default = "FF5733"
}

locals {
  red   = substr(var.hex_color, 0, 2)  # "FF"
  green = substr(var.hex_color, 2, 2)  # "57"
  blue  = substr(var.hex_color, 4, 2)  # "33"
}

output "color_channels" {
  value = {
    red   = local.red
    green = local.green
    blue  = local.blue
  }
}
```

## Step-by-Step Usage

1. Identify the start position (0-indexed) and length needed.
2. Call `substr(string, offset, length)`.
3. For "rest of string", use `-1` as length.
4. Test in `tofu console`:

```bash
tofu console

> substr("abcdef", 0, 3)
"abc"
> substr("abcdef", 3, -1)
"def"
> substr("abcdef", -3, 3)
"def"
```

## Negative Offsets

Negative offsets count from the end of the string:

```hcl
locals {
  s         = "hello-world"
  last_5    = substr(local.s, -5, 5)   # "world"
  last_char = substr(local.s, -1, 1)   # "d"
}
```

## substr vs split

| Use Case | Function |
|----------|----------|
| Extract by position | `substr` |
| Split by delimiter | `split` |
| Take n characters from start | `substr(s, 0, n)` |
| Get everything after nth char | `substr(s, n, -1)` |

## Conclusion

The `substr` function is the go-to tool in OpenTofu for positional string extraction. It is particularly useful for abbreviating names, extracting structured components like dates or color channels, and generating short identifiers from longer strings. Combined with `split` and other string functions, it gives you complete control over string parsing in your infrastructure configurations.
