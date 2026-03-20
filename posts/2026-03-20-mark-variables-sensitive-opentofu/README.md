# How to Mark Variables as Sensitive in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Security, Sensitive Variables, HCL, Secrets, Infrastructure as Code

Description: Learn how to mark OpenTofu variables as sensitive to prevent their values from appearing in plan output, apply output, and state file diffs.

---

The `sensitive = true` attribute on an input variable tells OpenTofu to redact its value from all output. This prevents passwords, API keys, and tokens from appearing in terminal output, CI/CD logs, or plan files. This guide covers marking variables sensitive and understanding where the values are still stored.

---

## Marking a Variable as Sensitive

```hcl
# variables.tf — sensitive variables

variable "database_password" {
  type        = string
  description = "PostgreSQL database password"
  sensitive   = true   # ← redacts value from all output
}

variable "api_key" {
  type        = string
  description = "External API key"
  sensitive   = true
}

variable "jwt_secret" {
  type      = string
  sensitive = true
}
```

---

## Effect on Plan Output

Without `sensitive = true`:
```
  + resource "aws_db_instance" "main" {
      + password = "my-secret-password-123"  # visible!
    }
```

With `sensitive = true`:
```
  + resource "aws_db_instance" "main" {
      + password = (sensitive value)  # redacted
    }
```

---

## Sensitive Variables Propagate Automatically

When a sensitive variable is used in a resource, the attribute that uses it also becomes sensitive:

```hcl
resource "aws_db_instance" "main" {
  identifier = "myapp-db"
  engine     = "postgres"
  username   = "admin"
  password   = var.database_password  # inherits sensitive marking

  # The 'password' attribute is now treated as sensitive throughout
}
```

---

## Sensitive Values Are Still Stored in State

Important: `sensitive = true` only affects **display**. The actual value is still stored in the state file.

```bash
# The value IS in the state file (base64 or plain text depending on backend)
# Always use a remote backend with encryption for sensitive state
tofu state show aws_db_instance.main
# password = (sensitive value)  ← shown redacted in CLI output

# But the raw state file contains the value
# cat terraform.tfstate | python3 -c "import sys,json; print(json.load(sys.stdin))"
# → contains actual password
```

Always use an encrypted remote state backend (S3 with SSE, HCP Terraform, etc.) when your state contains sensitive values.

---

## Sensitive Outputs

When a sensitive variable is used in an output, mark the output sensitive too:

```hcl
output "db_connection_string" {
  value     = "postgresql://admin:${var.database_password}@${aws_db_instance.main.endpoint}/app"
  sensitive = true  # required when value contains sensitive data
}
```

```bash
# Outputs are shown as (sensitive value)
tofu output db_connection_string
# (sensitive value)

# Access the value explicitly
tofu output -raw db_connection_string
# postgresql://admin:mypassword@db.host:5432/app
```

---

## Passing Sensitive Variables Securely

```bash
# Option 1: TF_VAR_ environment variable (recommended)
TF_VAR_database_password="my-secure-password" tofu apply

# Option 2: Read from a file (keep file out of version control)
tofu apply -var="database_password=$(cat ~/.secrets/db_password)"

# Option 3: Pass via tfvars file (not committed to git)
tofu apply -var-file="secrets.tfvars"
# secrets.tfvars: database_password = "my-secure-password"
```

---

## Summary

Mark variables as `sensitive = true` to prevent values from appearing in terminal output and CI/CD logs. The sensitive marking propagates to any output or resource attribute that uses the variable. Remember: sensitive variables still appear in plaintext in the state file — always use an encrypted remote state backend. For passing sensitive values, prefer `TF_VAR_` environment variables over `-var` flags to keep values out of shell history.
