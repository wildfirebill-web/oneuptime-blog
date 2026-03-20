# How to Prevent Secrets from Appearing in Plan Output in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Security, Sensitive Variables, Secrets, Infrastructure as Code, Best Practices

Description: Learn how to use OpenTofu's sensitive variable and output markings, along with provider-level suppression, to keep secrets out of plan and apply output.

## Introduction

By default, OpenTofu prints resource attribute values in plan output. Without explicit suppression, database passwords, API keys, and other credentials appear in terminal logs and CI/CD artifacts. OpenTofu provides several mechanisms to prevent this.

## Marking Variables as Sensitive

The simplest protection: mark the variable as `sensitive = true`:

```hcl
# variables.tf
variable "db_password" {
  type        = string
  description = "Database master password"
  sensitive   = true   # Redacts value in all plan/apply output
}

variable "api_key" {
  type      = string
  sensitive = true
}
```

When a sensitive variable is used in a resource, OpenTofu redacts its value:

```
# Plan output — password is hidden
  ~ resource "aws_db_instance" "main" {
      ~ password = (sensitive value)
    }
```

## Marking Outputs as Sensitive

```hcl
# outputs.tf
output "db_connection_string" {
  # Mark the output sensitive so it is redacted when shown
  value     = "postgresql://user:${var.db_password}@${aws_db_instance.main.address}:5432/mydb"
  sensitive = true
}
```

Sensitive outputs are still accessible programmatically but are masked in terminal output:

```
Outputs:

db_connection_string = <sensitive>
```

## Marking Resource Attributes as Sensitive (Provider-Level)

Many provider resources automatically mark sensitive attributes. For custom providers or overlooked attributes, you can wrap values with the `sensitive()` function:

```hcl
locals {
  # Explicitly mark a computed value as sensitive
  connection_url = sensitive("mysql://${var.db_user}:${var.db_password}@${aws_db_instance.main.address}/mydb")
}
```

## Preventing Sensitive Values in State (Write-Only Attributes)

OpenTofu 1.10+ supports write-only attributes that are never stored in state:

```hcl
resource "aws_iam_user_login_profile" "ops" {
  user = aws_iam_user.ops.name

  # Write-only: stored in the provider but never in state file
  pgp_key = var.pgp_public_key
  # password_reset_required is write-only as of provider v5.x
}
```

## Using Nonsensitive() When Downstream Logic Requires It

In some cases you may need to use a sensitive value in a non-sensitive context temporarily. Use `nonsensitive()` only after confirming the value cannot leak:

```hcl
# Only use nonsensitive() when you are certain the value is safe to display
output "db_host_with_port" {
  # Port and host are not secret; only the password was sensitive
  value = nonsensitive("${aws_db_instance.main.address}:${aws_db_instance.main.port}")
}
```

## Checking for Sensitive Value Leaks in CI

Add a post-plan check that fails if the word "password" appears unredacted in plan output:

```bash
#!/bin/bash
# ci-check.sh — fail if plan output contains unredacted secrets
PLAN_OUTPUT=$(tofu plan -no-color 2>&1)

# Check that passwords are redacted
if echo "$PLAN_OUTPUT" | grep -iE "password\s*=\s*\"[^(]"; then
  echo "ERROR: Unredacted password detected in plan output!"
  exit 1
fi
echo "Plan output looks clean."
```

## Conclusion

Mark all secret variables and outputs with `sensitive = true`, use the `sensitive()` function for computed values, and rely on write-only attributes for values that should never appear in state. These three layers together ensure secrets stay out of plan output, apply logs, and CI/CD artifacts.
