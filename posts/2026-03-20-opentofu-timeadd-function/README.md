# How to Use the timeadd Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the timeadd function in OpenTofu to add or subtract time durations from timestamps for expiry dates and scheduling.

## Introduction

The `timeadd` function in OpenTofu adds a duration to a timestamp, returning a new timestamp. It supports positive (future) and negative (past) durations, making it useful for computing expiry dates, certificate validity periods, and time-based scheduling.

## Syntax

```hcl
timeadd(timestamp, duration)
```

- **timestamp** - an RFC 3339 timestamp string
- **duration** - a duration string (e.g., `"24h"`, `"-30m"`, `"168h"`)
- Returns a new RFC 3339 timestamp

## Duration Formats

| Suffix | Meaning |
|--------|---------|
| `ns` | Nanoseconds |
| `us` | Microseconds |
| `ms` | Milliseconds |
| `s` | Seconds |
| `m` | Minutes |
| `h` | Hours |

Combine: `"1h30m"` = 1.5 hours

## Basic Examples

```hcl
output "one_day_later" {
  value = timeadd("2026-03-20T00:00:00Z", "24h")
  # Returns "2026-03-21T00:00:00Z"
}

output "one_hour_before" {
  value = timeadd("2026-03-20T14:00:00Z", "-1h")
  # Returns "2026-03-20T13:00:00Z"
}
```

## Practical Use Cases

### Certificate Validity Period

```hcl
locals {
  now          = timestamp()
  valid_hours  = 365 * 24  # 1 year
  expiry_time  = timeadd(local.now, "${local.valid_hours}h")
  expiry_date  = formatdate("YYYY-MM-DD", local.expiry_time)
}

resource "tls_self_signed_cert" "internal" {
  private_key_pem = tls_private_key.main.private_key_pem

  subject {
    common_name = "internal.example.com"
  }

  validity_period_hours = local.valid_hours
  allowed_uses          = ["key_encipherment", "digital_signature", "server_auth"]
}

output "cert_expiry" {
  value = local.expiry_date
}
```

### IAM User Access Key Expiry

```hcl
locals {
  key_expiry_date = formatdate(
    "YYYY-MM-DD",
    timeadd(timestamp(), "${90 * 24}h")  # 90 days
  )
}

resource "aws_iam_user" "service" {
  name = "service-account"

  tags = {
    Name         = "service-account"
    KeyExpiresOn = local.key_expiry_date
  }
}
```

### Maintenance Window Calculation

```hcl
variable "maintenance_start" {
  type    = string
  default = "2026-03-22T02:00:00Z"
}

locals {
  maintenance_duration = "2h"
  maintenance_end      = timeadd(var.maintenance_start, local.maintenance_duration)
}

output "maintenance_window" {
  value = {
    start = var.maintenance_start
    end   = local.maintenance_end
  }
}
```

### Temporary Resource TTL

```hcl
locals {
  ttl_hours   = 48
  created_at  = plantimestamp()
  expires_at  = timeadd(local.created_at, "${local.ttl_hours}h")
}

resource "aws_instance" "temp" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name      = "temp-instance"
    CreatedAt = local.created_at
    ExpiresAt = local.expires_at
    TTL       = "${local.ttl_hours}h"
  }
}
```

## Step-by-Step Usage

```bash
tofu console

> timeadd("2026-01-01T00:00:00Z", "720h")  # 30 days
"2026-01-31T00:00:00Z"
> timeadd("2026-03-20T12:00:00Z", "-30m")
"2026-03-20T11:30:00Z"
```

## Combining with timecmp

```hcl
locals {
  now     = timestamp()
  expiry  = "2026-06-01T00:00:00Z"
  warning = timeadd(expiry, "-168h")  # 7 days before expiry
  is_in_warning_window = timecmp(local.now, local.warning) >= 0
}
```

## Conclusion

The `timeadd` function enables time arithmetic in OpenTofu configurations. Use it to compute expiry dates, certificate validity periods, maintenance window end times, and any other scenario where you need to add or subtract a duration from a timestamp.
