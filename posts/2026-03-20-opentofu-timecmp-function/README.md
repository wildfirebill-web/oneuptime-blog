# How to Use the timecmp Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the timecmp function in OpenTofu to compare timestamps for expiry checking and time-based conditional configuration.

## Introduction

The `timecmp` function in OpenTofu compares two RFC 3339 timestamps and returns `-1`, `0`, or `1` depending on whether the first timestamp is before, equal to, or after the second. It enables time-based conditional logic in your configurations.

## Syntax

```hcl
timecmp(timestamp_a, timestamp_b)
```

- Returns `-1` if `a` is before `b`
- Returns `0` if `a` equals `b`
- Returns `1` if `a` is after `b`

## Basic Examples

```hcl
output "compare_times" {
  value = timecmp("2026-01-01T00:00:00Z", "2026-06-01T00:00:00Z")
  # Returns -1 (Jan is before June)
}

output "equal_times" {
  value = timecmp("2026-03-20T00:00:00Z", "2026-03-20T00:00:00Z")
  # Returns 0
}
```

## Practical Use Cases

### Checking Certificate Expiry

```hcl
variable "cert_expiry_date" {
  type        = string
  description = "Certificate expiry in RFC 3339 format"
  default     = "2026-12-31T00:00:00Z"
}

locals {
  now           = timestamp()
  is_expired    = timecmp(local.now, var.cert_expiry_date) >= 0
  warning_date  = timeadd(var.cert_expiry_date, "-168h")  # 7 days before
  in_warning    = timecmp(local.now, local.warning_date) >= 0
}

output "cert_status" {
  value = local.is_expired ? "EXPIRED" : (local.in_warning ? "EXPIRING_SOON" : "VALID")
}
```

### Enforcing Maintenance Window

```hcl
variable "maintenance_end" {
  type    = string
  default = "2026-04-01T06:00:00Z"
}

locals {
  is_past_maintenance = timecmp(timestamp(), var.maintenance_end) >= 0
}

resource "null_resource" "validate_deployment" {
  lifecycle {
    precondition {
      condition     = local.is_past_maintenance
      error_message = "Deployment not allowed until maintenance window ends at ${var.maintenance_end}."
    }
  }
}
```

### Time-Based Feature Flags

```hcl
variable "feature_launch_date" {
  type    = string
  default = "2026-04-01T00:00:00Z"
}

locals {
  feature_enabled = timecmp(timestamp(), var.feature_launch_date) >= 0
}

resource "aws_cloudfront_distribution" "app" {
  # ... distribution config
  
  dynamic "custom_error_response" {
    for_each = local.feature_enabled ? [] : [1]
    content {
      error_code         = 404
      response_code      = 302
      response_page_path = "/coming-soon"
    }
  }
}
```

### Subscription Expiry Check

```hcl
variable "license_expiry" {
  type = string
}

locals {
  is_license_valid = timecmp(timestamp(), var.license_expiry) < 0
}

resource "null_resource" "license_check" {
  lifecycle {
    precondition {
      condition     = local.is_license_valid
      error_message = "License expired on ${var.license_expiry}. Renew before deploying."
    }
  }
}
```

## Step-by-Step Usage

```bash
tofu console

> timecmp("2026-01-01T00:00:00Z", "2026-06-01T00:00:00Z")
-1
> timecmp(timestamp(), "2025-01-01T00:00:00Z")
1  # Current time is after 2025
```

## Converting to Boolean

```hcl
locals {
  # Is the current time after the target?
  is_after_target = timecmp(timestamp(), var.target_time) >= 0

  # Is current time before the deadline?
  is_before_deadline = timecmp(timestamp(), var.deadline) < 0
}
```

## Conclusion

The `timecmp` function enables time-based logic in OpenTofu — from certificate expiry checking to deployment scheduling and feature flag activation. Use it with `timestamp()`, `timeadd()`, and `precondition` blocks to implement time-aware infrastructure policies.
