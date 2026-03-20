# How to Use tflint with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, TFLint, Linting, Code Quality, Validation

Description: Learn how to set up and use tflint to lint OpenTofu configurations, catching errors, enforcing conventions, and validating provider-specific resource attributes beyond what tofu validate provides.

## Introduction

tflint is a linter specifically designed for Terraform and OpenTofu configurations. It catches issues that `tofu validate` cannot: invalid AWS instance types, deprecated attributes, undeclared variables used in expressions, and configurable naming convention violations. Running tflint before `tofu plan` prevents plan-time failures on clouds you haven't accessed yet.

## Installation

```bash
# macOS

brew install tflint

# Linux
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Windows (via Chocolatey)
choco install tflint

# Verify installation
tflint --version
```

## Initial Setup

```bash
# Create a .tflint.hcl configuration file
cat > .tflint.hcl << 'EOF'
config {
  format              = "default"
  call_module_type    = "local"
  disabled_by_default = false
}

plugin "aws" {
  enabled = true
  version = "0.32.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}
EOF

# Download configured plugins
tflint --init
```

## Basic Usage

```bash
# Lint the current directory
tflint

# Lint a specific directory
tflint --chdir=modules/vpc

# Lint recursively across all modules
tflint --recursive

# Show all findings with details
tflint --format=detailed

# Output JSON for CI parsing
tflint --format=json > tflint-output.json

# Exit with code 0 even on findings (for gradual adoption)
tflint --force
```

## What tflint Catches

```hcl
# ISSUE 1: Invalid EC2 instance type
resource "aws_instance" "web" {
  instance_type = "t2.nanooo"  # Error: invalid instance type
  ami           = "ami-12345"
}

# ISSUE 2: Deprecated attribute
resource "aws_s3_bucket" "logs" {
  bucket = "my-logs"
  acl    = "private"  # Warning: use aws_s3_bucket_acl resource instead
}

# ISSUE 3: Undeclared variable
resource "aws_instance" "web" {
  instance_type = var.instance_type  # Error: var.instance_type not declared
}

# ISSUE 4: Missing required provider
# terraform block without required_providers

# ISSUE 5: Naming convention violation (if configured)
resource "aws_s3_bucket" "MyBucket" {  # Should be snake_case
  bucket = "my-bucket"
}
```

## Plugin Configuration

```hcl
# .tflint.hcl - full configuration example
config {
  format              = "default"
  call_module_type    = "local"
  disabled_by_default = false
}

# AWS ruleset
plugin "aws" {
  enabled = true
  version = "0.32.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

# Azure ruleset
plugin "azurerm" {
  enabled = true
  version = "0.26.0"
  source  = "github.com/terraform-linters/tflint-ruleset-azurerm"
}

# Built-in Terraform rules
rule "terraform_comment_syntax" {
  enabled = true
}

rule "terraform_deprecated_index" {
  enabled = true
}

rule "terraform_required_providers" {
  enabled = true
}

rule "terraform_required_version" {
  enabled = true
}

rule "terraform_typed_variables" {
  enabled = true
}

rule "terraform_naming_convention" {
  enabled = true
  resource {
    format = "snake_case"
  }
}
```

## Ignoring Specific Issues

```hcl
# Ignore a specific rule for a file using annotations
# tflint-ignore: aws_instance_previous_type
resource "aws_instance" "legacy" {
  instance_type = "t2.micro"  # Must stay as t2 for legacy compatibility
}
```

```bash
# Ignore a specific rule globally
tflint --disable-rule=aws_instance_previous_type

# Ignore multiple rules
tflint --disable-rule=rule1 --disable-rule=rule2
```

## Pre-commit Integration

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/terraform-linters/tflint
    rev: v0.50.0
    hooks:
      - id: tflint
        name: tflint
        args: [--module]
```

## Makefile Integration

```makefile
.PHONY: lint

lint: lint-format lint-tflint

lint-format:
	tofu fmt -check -recursive .

lint-tflint:
	tflint --init
	tflint --recursive
```

## Conclusion

tflint catches a significant class of errors that only surface during `tofu plan` or `tofu apply` against real cloud providers - by running it locally or in CI before the plan, you get faster feedback without needing cloud credentials. The AWS plugin is essential for teams working with AWS as it validates instance types, RDS engines, and other AWS-specific values that OpenTofu itself cannot validate without calling the AWS API.
