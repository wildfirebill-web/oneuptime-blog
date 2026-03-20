# How to Use Sensitive Variables to Redact Values in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Sensitive Variables, Security, Secrets, Infrastructure as Code, Best Practices

Description: Learn how to declare input variables, locals, and outputs as sensitive in OpenTofu to automatically redact their values from plan and apply terminal output.

## Introduction

OpenTofu's `sensitive = true` attribute tells the engine to treat a value as a secret throughout its lifecycle. Sensitive values are replaced with `(sensitive value)` in all terminal output, log files, and JSON plan files, while remaining fully functional at the infrastructure level.

## Declaring a Sensitive Input Variable

```hcl
# variables.tf

variable "db_password" {
  type        = string
  description = "Master password for the RDS instance"
  sensitive   = true
}

variable "stripe_api_key" {
  type      = string
  sensitive = true
}

variable "tls_private_key" {
  type      = string
  sensitive = true
}
```

## Effect on Plan Output

When a sensitive variable is used in a resource, OpenTofu redacts it automatically:

```hcl
  # aws_db_instance.main will be updated in-place
  ~ resource "aws_db_instance" "main" {
        id       = "prod-postgres"
      ~ password = (sensitive value)
    }
```

## Sensitive Locals

If you compute a value derived from a sensitive variable, mark the local as sensitive too:

```hcl
locals {
  # Mark the local sensitive to propagate suppression downstream
  db_connection_string = sensitive(
    "postgresql://${var.db_user}:${var.db_password}@${aws_db_instance.main.address}:5432/mydb"
  )
}
```

## Sensitive Outputs

```hcl
output "db_password" {
  value     = var.db_password
  sensitive = true   # Required - OpenTofu errors if sensitive value used in non-sensitive output
}

output "connection_string" {
  value     = local.db_connection_string
  sensitive = true
}
```

Attempting to output a sensitive value without `sensitive = true` produces an error:

```text
Error: Output refers to sensitive values
  The output value is derived from a sensitive value, and so must also be declared as sensitive.
```

## Reading Sensitive Outputs Programmatically

Sensitive outputs are accessible despite being masked in the terminal:

```bash
# Read the sensitive output as JSON (value visible in JSON)
tofu output -json db_password

# Use in a script
DB_PASS=$(tofu output -raw db_password)
```

## Using sensitive() and nonsensitive() Functions

```hcl
# Wrap a known-sensitive value dynamically
locals {
  api_endpoint_with_key = sensitive("https://api.example.com?key=${var.api_key}")
}

# Unwrap a sensitive value when you are CERTAIN it is safe to display
# (e.g., the value is a public URL, not a secret)
output "public_url" {
  value = nonsensitive(var.public_cdn_url)
}
```

## Variable Validation Still Works on Sensitive Variables

```hcl
variable "db_password" {
  type      = string
  sensitive = true

  validation {
    # Validation runs on the actual value, not the redacted form
    condition     = length(var.db_password) >= 16
    error_message = "Database password must be at least 16 characters."
  }
}
```

## Conclusion

Marking variables and outputs as `sensitive = true` is the simplest and most effective way to keep secrets out of OpenTofu's terminal output and JSON plans. Combined with the `sensitive()` function for computed values and `nonsensitive()` for intentional unwrapping, you have fine-grained control over what appears in logs without sacrificing functionality.
