# How to Use Write-Only Attributes to Protect Credentials in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Write-Only Attributes, Security, Credential, Infrastructure as Code, State

Description: Learn how OpenTofu's write-only attribute feature prevents sensitive values from ever being stored in state files, providing the strongest protection for credentials.

## Introduction

Even when a variable is marked `sensitive = true`, its value is still stored in the OpenTofu state file in plaintext. Write-only attributes (introduced in OpenTofu 1.10) solve this: they are sent to the provider on creation/update but never stored in state, eliminating the risk of credential exposure through a compromised state file.

## Understanding Write-Only vs Sensitive

| Feature | `sensitive = true` | Write-Only |
|---|---|---|
| Hidden in plan output | Yes | Yes |
| Stored in state file | Yes (encrypted only if backend encrypts) | No |
| Available in `tofu state show` | Yes (masked) | No |
| Re-read on refresh | Yes | No (provider must accept it) |

## Using Write-Only Attributes

Write-only attributes are defined by the provider, not by you. You simply assign them as you would any attribute - the provider handles them specially:

```hcl
# Example: creating an IAM user login profile

# The password attribute is write-only in the AWS provider
resource "aws_iam_user_login_profile" "ops" {
  user                    = aws_iam_user.ops.name
  # This value is sent to AWS but NEVER stored in state
  password_reset_required = true
}

# Example: setting a database password as write-only
resource "aws_db_instance" "main" {
  identifier        = "prod-postgres"
  engine            = "postgres"
  instance_class    = "db.t3.medium"
  allocated_storage = 20

  username = "admin"
  # In AWS provider v5.x+, password is a write-only attribute
  password = var.db_password   # var must be sensitive = true

  skip_final_snapshot = false
}
```

## Ephemeral Resources (OpenTofu 1.10+)

Ephemeral resources fetch values that are used during the apply but never written to state:

```hcl
# An ephemeral resource provides values only during apply - not stored in state
ephemeral "aws_secretsmanager_secret_version" "db_pass" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  username = "admin"
  # Reference the ephemeral value - never stored in state
  password = ephemeral.aws_secretsmanager_secret_version.db_pass.secret_string
}
```

## Checking Whether an Attribute is Write-Only

Provider documentation marks write-only attributes explicitly. You can also check at the CLI:

```bash
# Show the schema for a resource type and look for write_only: true
tofu providers schema -json | \
  jq '.provider_schemas["registry.opentofu.org/hashicorp/aws"].resource_schemas["aws_db_instance"].block.attributes | to_entries[] | select(.value.write_only == true) | .key'
```

## Combining Write-Only with Sensitive Variables

```hcl
variable "db_password" {
  type      = string
  sensitive = true   # Redacted in output AND sourced from a secret
}

resource "aws_db_instance" "main" {
  username = "admin"
  # If the provider supports write-only for password, it won't appear in state
  password = var.db_password
}
```

## Lifecycle: Handling Write-Only on Updates

Because write-only values are not stored in state, OpenTofu cannot detect changes to them automatically. Use `lifecycle.replace_triggered_by` if you need to force replacement when the password changes:

```hcl
resource "aws_db_instance" "main" {
  username = "admin"
  password = var.db_password

  lifecycle {
    # If the password variable changes, recreate the resource
    replace_triggered_by = [var.db_password]
  }
}
```

## Conclusion

Write-only attributes and ephemeral resources represent OpenTofu's most advanced secret protection: credentials that flow to the cloud provider during apply but leave no trace in the state file. As provider support for write-only attributes grows, teams should prefer them over sensitive variables alone for the highest-value credentials like database master passwords and API keys.
