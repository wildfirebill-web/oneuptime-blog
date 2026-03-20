# How to Test OpenTofu Configurations with tflint

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, tflint, Linting, Code Quality, Best Practices

Description: Learn how to use tflint to lint OpenTofu configurations, catch deprecated syntax, enforce naming conventions, and validate provider-specific best practices before applying.

## Introduction

tflint is a linter for OpenTofu/Terraform that catches errors not detectable by `tofu validate` — deprecated resource arguments, invalid instance types, naming convention violations, and provider-specific issues. It's faster than running `tofu plan` and works without cloud credentials.

## Installation and Configuration

```bash
# Install tflint
brew install tflint  # macOS
# or
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Install AWS ruleset
tflint --init
```

```hcl
# .tflint.hcl (project root)
config {
  format              = "default"
  call_module_type    = "local"
  force               = false
  disabled_by_default = false
}

plugin "aws" {
  enabled = true
  version = "0.32.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

# Enable specific rules
rule "terraform_naming_convention" {
  enabled = true

  resource {
    format = "snake_case"
  }

  variable {
    format = "snake_case"
  }

  output {
    format = "snake_case"
  }
}

rule "terraform_required_providers" {
  enabled = true
}

rule "terraform_required_version" {
  enabled = true
}

rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}
```

## Running tflint

```bash
# Run tflint in the current directory
tflint

# Run recursively across all modules
tflint --recursive

# Run with specific format
tflint --format=compact

# Show all rules (including disabled)
tflint --format=json > tflint-results.json

# Initialize plugins before first run
tflint --init
```

## Common tflint Findings

```hcl
# FINDING: aws_instance_invalid_type
resource "aws_instance" "web" {
  # tflint catches invalid instance types before applying
  instance_type = "t2.xlarge"  # Warning: consider t3.xlarge for better performance
}

# FINDING: terraform_deprecated_interpolation
output "example" {
  # Deprecated interpolation syntax
  value = "${var.example}"  # Use: value = var.example
}

# FINDING: terraform_naming_convention
resource "aws_s3_bucket" "MyBucket" {  # Should be snake_case: my_bucket
  bucket = "my-bucket"
}

# FINDING: terraform_documented_variables
variable "instance_type" {
  # Missing description
  type = string
}
```

## Provider-Specific Rules

```bash
# AWS ruleset checks include:
# aws_instance_invalid_type - validates EC2 instance types
# aws_db_instance_invalid_engine - validates RDS engine names
# aws_iam_policy_sid_invalid_characters - validates IAM policy SID format
# aws_elasticache_cluster_invalid_type - validates ElastiCache node types

tflint --only=aws_instance_invalid_type,aws_db_instance_invalid_engine
```

## Custom Rules

```go
// rules/check_s3_bucket_prefix.go
package rules

import (
    "github.com/terraform-linters/tflint-plugin-sdk/hclext"
    "github.com/terraform-linters/tflint-plugin-sdk/tflint"
)

type AwsS3BucketPrefixRule struct {
    tflint.DefaultRule
}

func (r *AwsS3BucketPrefixRule) Name() string {
    return "aws_s3_bucket_required_prefix"
}

func (r *AwsS3BucketPrefixRule) Enabled() bool {
    return true
}

func (r *AwsS3BucketPrefixRule) Severity() tflint.Severity {
    return tflint.WARNING
}
```

## CI/CD Integration

```yaml
# .github/workflows/tflint.yml
name: tflint

on: [pull_request]

jobs:
  tflint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup tflint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: v0.50.0

      - name: Init tflint plugins
        run: tflint --init

      - name: Run tflint
        run: tflint --recursive --format=compact
```

## Ignoring Specific Rules per File

```hcl
# main.tf
# tflint-ignore: terraform_naming_convention
resource "aws_s3_bucket" "S3Bucket" {
  bucket = "legacy-bucket-name"
}
```

## Conclusion

tflint catches a class of errors that `tofu validate` misses — provider-specific invalid values, deprecated syntax, and convention violations. Run it in pre-commit hooks for immediate developer feedback and as a required CI check. The recursive mode (`--recursive`) lints all modules in a monorepo with a single command, making it practical for large infrastructure codebases.
