# How to Use tofu.applying for Plan vs Apply Differentiation in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, tofu.applying, Ephemeral, Plan, Apply, HCL, Infrastructure as Code

Description: Learn how to use the tofu.applying built-in value in OpenTofu to differentiate behavior between plan and apply phases within ephemeral expressions.

---

`tofu.applying` is a built-in boolean value available in ephemeral expressions in OpenTofu 1.10+. It evaluates to `true` during `tofu apply` and `false` during `tofu plan`. This lets you write logic that behaves differently at plan time versus apply time.

---

## What tofu.applying Returns

```hcl
# tofu.applying is:
# - false during: tofu plan
# - true during:  tofu apply

# It can only be used in ephemeral contexts
ephemeral "aws_ssm_parameter" "config" {
  name = tofu.applying ? "/production/config" : "/staging/config"
  # Uses staging config during plan, production during apply
}
```

---

## Common Use Case: Avoiding Expensive Operations During Plan

```hcl
ephemeral "aws_secretsmanager_secret_version" "db_password" {
  # Only fetch the actual secret during apply — use a placeholder during plan
  secret_id = tofu.applying ? "production/database/password" : null

  # During plan, this will be null (no API call)
  # During apply, it fetches the real secret
}
```

Note: Using `null` for `secret_id` during plan will cause the ephemeral resource to return null values, which is expected.

---

## Differentiating Log Verbosity

```hcl
# Use a more verbose/debug config during plan for validation
# Use the real config during apply

locals {
  ephemeral log_level = tofu.applying ? "warn" : "debug"
}

resource "null_resource" "configure" {
  provisioner "local-exec" {
    command     = "./configure.sh"
    environment = {
      LOG_LEVEL = local.log_level   # ephemeral local
    }
  }
}
```

---

## Using Different Credential Sources

```hcl
# During plan: use read-only credentials for validation
# During apply: use full-access credentials for deployment

ephemeral "aws_iam_role" "deploy_role" {
  role_arn = tofu.applying ? (
    "arn:aws:iam::123456789012:role/DeployRole"
  ) : (
    "arn:aws:iam::123456789012:role/ReadOnlyRole"
  )
  session_name = "opentofu-operation"
}

provider "aws" {
  alias      = "deployment"
  access_key = ephemeral.aws_iam_role.deploy_role.access_key_id
  secret_key = ephemeral.aws_iam_role.deploy_role.secret_access_key
  token      = ephemeral.aws_iam_role.deploy_role.session_token
}
```

---

## Avoiding Side Effects During Plan

When using the `external` data source or ephemeral resources that have side effects (like issuing Vault tokens or creating STS sessions), `tofu.applying` lets you skip those side effects during planning:

```hcl
# Only create a Vault lease during apply, not plan
ephemeral "vault_database_secret" "creds" {
  count = tofu.applying ? 1 : 0

  mount = "database"
  name  = "production-role"
}

resource "null_resource" "run_migration" {
  provisioner "local-exec" {
    command = "migrate.sh"
    environment = {
      DB_USER = tofu.applying ? ephemeral.vault_database_secret.creds[0].username : ""
      DB_PASS = tofu.applying ? ephemeral.vault_database_secret.creds[0].password : ""
    }
  }
}
```

---

## tofu.applying in Preconditions

```hcl
ephemeral "aws_secretsmanager_secret_version" "config" {
  secret_id = "production/app-config"

  lifecycle {
    precondition {
      # During plan, skip validation that requires the actual secret
      # During apply, enforce the check
      condition     = !tofu.applying || length(self.secret_string) > 0
      error_message = "App config secret must not be empty."
    }
  }
}
```

---

## Limitations

- `tofu.applying` is only available in ephemeral expression contexts
- It cannot be used in regular resource arguments, outputs, or non-ephemeral locals
- It's designed specifically for ephemeral resource configurations and ephemeral variable expressions

---

## Summary

`tofu.applying` is a built-in boolean (OpenTofu 1.10+) that is `true` during apply and `false` during plan. Use it in ephemeral contexts to skip expensive API calls during planning, differentiate credential scopes between plan and apply, and avoid side effects from ephemeral resource evaluation at plan time. This improves plan performance and reduces unnecessary credential usage.
