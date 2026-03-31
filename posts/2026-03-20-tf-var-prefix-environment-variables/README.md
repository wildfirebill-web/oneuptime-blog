# How to Set Variables Using Environment Variables with TF_VAR_ Prefix (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Variable, Environment Variable, TF_VAR_, CI/CD, DevOps

Description: Learn how to use the TF_VAR_ prefix to set OpenTofu variable values via environment variables for CI/CD pipelines and secret injection.

---

Any environment variable prefixed with `TF_VAR_` is automatically read by OpenTofu as a variable value. This is the most secure way to pass sensitive values like passwords and API keys - they never appear in command history, process listings, or log files.

---

## Basic TF_VAR_ Usage

```bash
# Set variables via environment variables

export TF_VAR_region="us-east-1"
export TF_VAR_instance_count="3"
export TF_VAR_environment="production"

# OpenTofu reads them automatically - no -var flags needed
tofu plan
tofu apply
```

The environment variable name maps to the variable name:
- `TF_VAR_region` → variable `region`
- `TF_VAR_instance_count` → variable `instance_count`
- `TF_VAR_database_password` → variable `database_password`

---

## Passing Sensitive Values Securely

```bash
# INSECURE: password visible in shell history
tofu apply -var="database_password=mysecret"

# SECURE: use TF_VAR_ - doesn't appear in history
export TF_VAR_database_password="mysecret"
tofu apply

# Even more secure: don't export, pass only for the single command
TF_VAR_database_password="mysecret" tofu apply
```

---

## TF_VAR_ in CI/CD Pipelines

```yaml
# GitHub Actions - pass secrets via TF_VAR_
- name: Deploy infrastructure
  env:
    TF_VAR_database_password: ${{ secrets.DB_PASSWORD }}
    TF_VAR_api_key: ${{ secrets.API_KEY }}
    TF_VAR_region: ${{ vars.AWS_REGION }}  # non-secret var
    TF_VAR_environment: "production"
  run: |
    tofu apply -auto-approve
    # All TF_VAR_ env vars are automatically used
```

```yaml
# GitLab CI - same approach
deploy:
  variables:
    TF_VAR_environment: "production"
  script:
    - export TF_VAR_database_password="$DB_PASSWORD"  # from GitLab CI/CD variables
    - tofu apply -auto-approve
```

---

## Complex Types via TF_VAR_

For complex types, use JSON syntax in the environment variable value:

```bash
# Pass a list
export TF_VAR_allowed_ports='[80, 443, 22]'

# Pass a map
export TF_VAR_tags='{"Environment":"prod","Team":"platform"}'

# Pass an object
export TF_VAR_server_config='{"instance_type":"t3.large","disk_size_gb":100}'

tofu plan
```

---

## Variable Precedence Including TF_VAR_

OpenTofu evaluates variables in this order (highest wins):

```text
1. -var flags on the command line (highest priority)
2. -var-file flags on the command line
3. *.auto.tfvars files (alphabetically)
4. terraform.tfvars
5. TF_VAR_ environment variables
6. Variable defaults in configuration (lowest priority)
```

```bash
# Example: TF_VAR_ is overridden by -var flag
export TF_VAR_region="us-east-1"
tofu plan -var="region=us-west-2"
# Region used: us-west-2 (-var wins)
```

---

## Listing Active TF_VAR_ Variables

```bash
# See all TF_VAR_ variables currently set in your environment
env | grep ^TF_VAR_

# Output:
# TF_VAR_region=us-east-1
# TF_VAR_environment=production
# TF_VAR_database_password=[hidden]
```

---

## Summary

The `TF_VAR_` prefix is the recommended way to pass secrets and CI/CD-specific variable values to OpenTofu. Values set this way never appear in shell history or process tables, and CI/CD platforms like GitHub Actions and GitLab CI inject them from their secret stores automatically. For complex types (lists, maps, objects), use JSON syntax in the environment variable value.
