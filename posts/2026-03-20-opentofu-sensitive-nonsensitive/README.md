# How to Use the sensitive and nonsensitive Functions in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the sensitive and nonsensitive functions in OpenTofu to control the redaction of sensitive values in plan output and logs.

## Introduction

The `sensitive` and `nonsensitive` functions in OpenTofu control whether a value is treated as sensitive (redacted in plan output) or non-sensitive (displayed normally). `sensitive()` marks a value as sensitive, while `nonsensitive()` removes the sensitive marking when you intentionally want to reveal a value.

## Syntax

```hcl
sensitive(value)
nonsensitive(value)
```

- `sensitive(value)` — marks the value as sensitive (redacted in output)
- `nonsensitive(value)` — removes the sensitive marking

## Basic Examples

```hcl
output "password" {
  sensitive = true
  value     = var.db_password
}

# Using sensitive() function to mark a computed value
locals {
  derived_token = sensitive("${var.prefix}-${var.secret_key}")
}
```

## Practical Use Cases

### Marking Computed Secrets as Sensitive

```hcl
variable "app_name" {
  type    = string
  default = "myapp"
}

variable "secret_key" {
  type      = string
  sensitive = true
}

locals {
  # The derived value inherits sensitivity, but we can be explicit
  api_key = sensitive("${var.app_name}:${var.secret_key}")
}

resource "aws_ssm_parameter" "api_key" {
  name      = "/app/api-key"
  type      = "SecureString"
  value     = local.api_key
}
```

### Protecting Combined Credentials

```hcl
variable "db_username" {
  type    = string
  default = "appuser"
}

variable "db_password" {
  type      = string
  sensitive = true
}

locals {
  # Combine into connection string — mark as sensitive
  connection_string = sensitive(
    "postgresql://${var.db_username}:${var.db_password}@${aws_db_instance.main.endpoint}/${var.db_name}"
  )
}

resource "aws_ssm_parameter" "db_conn" {
  name      = "/app/db-connection"
  type      = "SecureString"
  value     = local.connection_string
}
```

### Using nonsensitive for Debugging

```hcl
variable "debug_mode" {
  type    = bool
  default = false
}

output "db_host" {
  # Reveal the host (not sensitive) even if other DB values are sensitive
  value = nonsensitive(aws_db_instance.main.address)
}
```

### Selectively Revealing Sensitive-Marked Values

```hcl
data "aws_secretsmanager_secret_version" "app" {
  secret_id = aws_secretsmanager_secret.app.id
}

locals {
  secret_data = jsondecode(data.aws_secretsmanager_secret_version.app.secret_string)
  # The whole secret_data is sensitive, but we may want to expose the non-secret hostname
  db_host = nonsensitive(local.secret_data["db_host"])
}

output "db_host" {
  value = local.db_host  # Not marked sensitive — will show in plan
}
```

### Sensitive Output Pattern

```hcl
output "rds_password" {
  description = "RDS master password"
  sensitive   = true
  value       = aws_db_instance.main.password
}

# In module output:
output "connection_info" {
  sensitive = true
  value = {
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    password = aws_db_instance.main.password
  }
}
```

## Step-by-Step Usage

1. Mark sensitive outputs with `sensitive = true` in the output block.
2. Use `sensitive()` to mark computed values as sensitive.
3. Use `nonsensitive()` only when you are certain a value does not need redaction.

## Important Notes

- Sensitive values are still stored in state (unredacted).
- They are only hidden from plan/apply console output.
- Use proper state encryption (e.g., S3 backend with KMS) for true security.

## Conclusion

The `sensitive` and `nonsensitive` functions give fine-grained control over OpenTofu's output redaction. Use `sensitive()` to protect derived secrets, mark computed credentials, and ensure connection strings don't appear in logs. Use `nonsensitive()` carefully and only for non-secret fields that happen to be part of a sensitive structure.
