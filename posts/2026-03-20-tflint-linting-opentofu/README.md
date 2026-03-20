# How to Use tflint for Linting OpenTofu Configurations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, tflint, Linting, Code Quality, Infrastructure as Code, DevOps

Description: Learn how to use tflint to catch configuration errors, deprecated attributes, and provider-specific best practice violations in OpenTofu configurations before planning.

## Introduction

`tflint` goes beyond `tofu validate` — while validate catches syntax errors, tflint catches semantic issues: deprecated attributes, invalid instance types, missing required module arguments, and provider-specific configuration mistakes. It integrates with provider plugins to know what valid values look like.

## Installing tflint

```bash
# macOS
brew install tflint

# Linux
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Docker
docker pull ghcr.io/terraform-linters/tflint-bundle:latest

tflint --version
```

## Installing Provider Plugins

tflint's provider plugins enable provider-specific rule checking:

```bash
# Initialize tflint with provider plugins
tflint --init

# Or specify explicitly
tflint --init --config .tflint.hcl
```

## .tflint.hcl Configuration

```hcl
# .tflint.hcl
config {
  # Use OpenTofu module mode
  module = true
  # Force exit 1 on violations
  force = false
}

# AWS provider rules
plugin "aws" {
  enabled    = true
  version    = "0.31.0"
  source     = "github.com/terraform-linters/tflint-ruleset-aws"
}

# Google provider rules
plugin "google" {
  enabled = true
  version = "0.28.0"
  source  = "github.com/terraform-linters/tflint-ruleset-google"
}

# Built-in rules
rule "terraform_deprecated_interpolation" {
  enabled = true
}

rule "terraform_naming_convention" {
  enabled = true

  variable {
    format = "snake_case"
  }

  resource {
    format = "snake_case"
  }
}

rule "terraform_required_version" {
  enabled = true
}

rule "terraform_documented_variables" {
  enabled = true
}
```

## Running tflint

```bash
# Lint the current directory
tflint

# Lint with specific config
tflint --config .tflint.hcl

# Lint all modules recursively
tflint --recursive

# Show only errors (not warnings)
tflint --minimum-failure-severity=error

# Format output as JSON
tflint --format=json
```

## Common tflint Findings

```hcl
# VIOLATION: aws_instance_invalid_type
# Error: "t2.nano" is an invalid value as instance_type
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.nano"   # Invalid — use t3.nano
}

# FIX
resource "aws_instance" "web" {
  instance_type = "t3.nano"
}

# VIOLATION: terraform_naming_convention
# Error: Resource name "WebServer" must match snake_case format
resource "aws_instance" "WebServer" {}   # Should be "web_server"

# VIOLATION: terraform_documented_variables
# Error: Variable "db_password" does not have a description
variable "db_password" {
  type = string
  # Missing: description = "..."
}
```

## GitHub Actions Integration

```yaml
- name: tflint
  uses: reviewdog/action-tflint@master
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    reporter: github-pr-review
    flags: "--recursive --module"
    tflint_rulesets: "aws google"
```

## Pre-Commit Hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.6
    hooks:
      - id: terraform_tflint
        args:
          - --args=--config=.tflint.hcl
```

## Conclusion

tflint catches issues that `tofu validate` misses — invalid cloud resource configurations, deprecated syntax, and naming convention violations. Install provider plugins to get provider-specific rule checking, configure naming convention rules to enforce your team's standards, and run tflint in pre-commit hooks and CI to prevent these issues from reaching code review.
