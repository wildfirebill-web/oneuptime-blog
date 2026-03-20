# How to Use the format Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the format function in OpenTofu to create formatted strings with printf-style placeholders for precise resource naming and output generation.

## Introduction

The `format` function in OpenTofu creates formatted strings using printf-style format specifiers. It gives you more control over string construction than simple interpolation, supporting padding, number formatting, and type-specific representations.

## Syntax

```hcl
format(spec, values...)
```

- **spec** — a format string with `%` placeholders
- **values** — one argument per placeholder
- Returns the formatted string

## Format Specifiers

| Specifier | Meaning |
|-----------|---------|
| `%s` | String |
| `%d` | Decimal integer |
| `%f` | Floating-point |
| `%e` | Scientific notation |
| `%g` | Compact float |
| `%b` | Binary |
| `%o` | Octal |
| `%x` | Hex (lowercase) |
| `%X` | Hex (uppercase) |
| `%v` | Default representation |
| `%%` | Literal percent sign |

## Basic Examples

```hcl
output "string_format" {
  value = format("Hello, %s!", "world")           # Returns "Hello, world!"
}

output "padded_number" {
  value = format("%05d", 42)                       # Returns "00042"
}

output "float_precision" {
  value = format("%.2f", 3.14159)                  # Returns "3.14"
}

output "hex_format" {
  value = format("%x", 255)                        # Returns "ff"
}
```

## Practical Use Cases

### Zero-Padded Resource Numbers

```hcl
variable "instance_count" {
  type    = number
  default = 5
}

locals {
  # Create numbered instance names with zero padding: node-001, node-002...
  instance_names = [
    for i in range(var.instance_count) :
    format("node-%03d", i + 1)
  ]
}

output "instance_names" {
  value = local.instance_names
  # ["node-001", "node-002", "node-003", "node-004", "node-005"]
}
```

### Generating Port Ranges

```hcl
variable "base_port" {
  type    = number
  default = 8080
}

variable "service_count" {
  type    = number
  default = 4
}

locals {
  service_configs = [
    for i in range(var.service_count) :
    {
      name = format("service-%d", i + 1)
      port = var.base_port + i
    }
  ]
}
```

### Formatting Float Percentages

```hcl
variable "cpu_threshold_fraction" {
  type    = number
  default = 0.7567
}

resource "aws_cloudwatch_metric_alarm" "cpu" {
  alarm_name        = "high-cpu"
  alarm_description = format("CPU exceeds %.1f%%", var.cpu_threshold_fraction * 100)

  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  statistic           = "Average"
  period              = 60
  threshold           = var.cpu_threshold_fraction * 100
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
}
```

### Hex Resource Identifiers

```hcl
variable "environment_id" {
  type    = number
  default = 255
}

locals {
  # Represent environment as 2-digit hex
  env_hex = format("%02x", var.environment_id)
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name  = "vpc-${local.env_hex}"
    EnvId = tostring(var.environment_id)
  }
}
```

### CIDR Block Generation

```hcl
variable "vpc_prefix" {
  type    = number
  default = 10
}

locals {
  # Generate CIDR blocks with format
  vpc_cidr   = format("%d.0.0.0/8", var.vpc_prefix)
  subnet_cidr = format("%d.1.0.0/24", var.vpc_prefix)
}

output "vpc_cidr" {
  value = local.vpc_cidr  # "10.0.0.0/8"
}
```

## Step-by-Step Usage

1. Write the format string with `%` placeholders.
2. Pass the values as additional arguments.
3. Test in `tofu console`:

```bash
tofu console

> format("%-10s: %d", "count", 42)
"count     : 42"
> format("%08.3f", 3.14)
"0003.140"
```

## Padding and Alignment

```hcl
locals {
  # Right-align in 10-character field
  right_aligned = format("%10s", "hello")  # "     hello"

  # Left-align in 10-character field
  left_aligned  = format("%-10s", "hello")  # "hello     "

  # Zero-pad to 6 digits
  zero_padded   = format("%06d", 42)  # "000042"
}
```

## format vs String Interpolation

Use `format` when you need:
- Numeric formatting (padding, precision, hex)
- Multiple type conversions in one string
- Printf-style precision

Use `"${var}"` interpolation for simple string concatenation.

## Conclusion

The `format` function in OpenTofu bridges the gap between simple string interpolation and sophisticated string building. It is essential for zero-padded identifiers, float precision formatting, hex representations, and column-aligned output. Master `format` to produce clean, consistent resource names and descriptions in your infrastructure code.
