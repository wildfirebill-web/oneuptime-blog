# How to Use tofu validate in Pre-Commit Hooks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Tofu validate, Pre-Commit, Validation, Syntax Checking

Description: Learn how to run tofu validate in pre-commit hooks to catch syntax errors, undefined references, and type mismatches in OpenTofu configurations before they reach CI/CD.

## Introduction

`tofu validate` checks OpenTofu configuration syntax and internal consistency without accessing remote resources or cloud APIs. Running it as a pre-commit hook catches typos, missing required arguments, undefined variables, and type errors before they waste CI/CD pipeline time.

## tofu validate Basics

```bash
# Initialize first (required for validate to work)

tofu init -backend=false  # Skip backend configuration for validation

# Validate the current directory
tofu validate

# Validate and output JSON (for CI parsing)
tofu validate -json

# Validate with compact output
tofu validate -no-color

# Exit codes:
# 0 = valid configuration
# 1 = error or invalid configuration
```

## What tofu validate Catches

```hcl
# CATCH 1: Missing required argument
resource "aws_instance" "web" {
  # Missing required 'ami' argument
  instance_type = "t3.medium"
}
# Error: Missing required argument "ami"

# CATCH 2: Undefined variable reference
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type  # var not declared
}
# Error: Reference to undeclared input variable "instance_type"

# CATCH 3: Type mismatch
variable "count" {
  type = number
}
resource "aws_instance" "web" {
  count = "three"  # String, not number
}
# Error: Invalid value for input variable

# CATCH 4: Syntax errors
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket
  # Missing closing quote
}

# CATCH 5: Invalid attribute name
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_typo = "t3.medium"  # Typo in attribute name
}
# Error: An argument named "instance_typo" is not expected here
```

## Pre-commit Hook for tofu validate

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: tofu-init-validate
        name: OpenTofu Init and Validate
        language: system
        entry: bash
        args:
          - -c
          - |
            set -e
            for dir in $(find . -name "*.tf" -not -path "*/.terraform/*" | xargs dirname | sort -u); do
              echo "Validating $dir..."
              (cd "$dir" && tofu init -backend=false -input=false -no-color > /dev/null && tofu validate -no-color)
            done
        files: \.tf$
        pass_filenames: false
```

## Efficient Validation (Only Changed Modules)

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: tofu-validate-changed
        name: OpenTofu Validate Changed Files
        language: system
        entry: bash
        args:
          - -c
          - |
            set -e
            # Get unique directories of changed .tf files
            DIRS=$(echo "$@" | tr ' ' '\n' | xargs dirname | sort -u)
            for dir in $DIRS; do
              echo "Validating $dir"
              (cd "$dir" && tofu init -backend=false -input=false -no-color > /dev/null && tofu validate -no-color)
            done
        files: \.tf$
        # Pass changed .tf files as arguments
```

## Using antonbabenko/pre-commit-terraform

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.92.0
    hooks:
      - id: terraform_validate
        name: OpenTofu Validate
        args:
          - --hook-config=--retry-once-with-cleanup=true
          # Override terraform binary with tofu
          - --tf-init-args=-backend=false
```

## CI/CD Integration

```yaml
# .github/workflows/validate.yml
name: Validate

on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        module:
          - modules/vpc
          - modules/database
          - modules/ecs-service
          - environments/prod
          - environments/staging

    steps:
      - uses: actions/checkout@v4
      - uses: opentofu/setup-opentofu@v1

      - name: Init ${{ matrix.module }}
        run: tofu -chdir=${{ matrix.module }} init -backend=false

      - name: Validate ${{ matrix.module }}
        run: tofu -chdir=${{ matrix.module }} validate
```

## Handling Backend Configuration

```bash
# When running validate locally without backend credentials:
tofu init -backend=false

# When the module requires variables:
tofu validate
# Passes even if required variables don't have values yet
# (validate doesn't execute variable validation conditions)

# For modules with complex variable dependencies:
cat > /tmp/test.tfvars << 'EOF'
environment = "test"
vpc_cidr    = "10.0.0.0/16"
EOF
tofu validate -var-file=/tmp/test.tfvars
```

## Conclusion

`tofu validate` in pre-commit hooks provides a first line of defense against syntax and reference errors before they reach CI/CD. The key challenge is handling `tofu init` - you need it to download provider schemas for proper validation, but the `-backend=false` flag skips the backend configuration that requires credentials. Combine it with `tofu fmt -check` in the same hook set for a complete local validation workflow.
