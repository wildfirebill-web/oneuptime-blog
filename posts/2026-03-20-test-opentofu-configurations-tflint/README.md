# How to Lint OpenTofu Configurations with TFLint

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, TFLint, Linting, Code Quality, Infrastructure as Code

Description: Learn how to use TFLint to catch errors, enforce best practices, and detect provider-specific issues in OpenTofu configurations before running tofu plan.

## Introduction

TFLint is a linter for Terraform and OpenTofu that catches errors not detected by `tofu validate`, such as invalid instance types, deprecated parameters, and module best-practice violations. It supports provider-specific rules through a plugin system.

## Installing TFLint

On macOS:

```bash
brew install tflint
```

On Linux:

```bash
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
```

Verify:

```bash
tflint --version
```

## Initializing TFLint with Plugins

Create a `.tflint.hcl` configuration file:

```hcl
plugin "aws" {
  enabled = true
  version = "0.30.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

plugin "azurerm" {
  enabled = true
  version = "0.26.0"
  source  = "github.com/terraform-linters/tflint-ruleset-azurerm"
}
```

Initialize plugins:

```bash
tflint --init
```

## Running TFLint

```bash
# Lint current directory

tflint

# Lint specific directory
tflint --chdir=./modules/networking

# Recursive scan
tflint --recursive
```

## Common Issues Caught by TFLint

```bash
# Invalid EC2 instance type
resource "aws_instance" "web" {
  instance_type = "t2.invalidtype"  # TFLint catches this
}

# Deprecated resource
resource "aws_alb" "web" {  # Should use aws_lb
  ...
}

# Missing required tags
resource "aws_s3_bucket" "logs" {
  # TFLint can enforce required tags via custom rules
}
```

## Enabling Specific Rules

```hcl
# .tflint.hcl
rule "aws_instance_invalid_type" {
  enabled = true
}

rule "aws_instance_previous_type" {
  enabled = true
}

rule "terraform_deprecated_interpolation" {
  enabled = true
}
```

## Ignoring Rules Inline

```hcl
resource "aws_instance" "legacy" {
  instance_type = "t1.micro"  # tflint-ignore: aws_instance_previous_type
}
```

## CI/CD Integration

```yaml
- name: TFLint
  run: |
    tflint --init
    tflint --recursive
```

GitHub Actions:

```yaml
- uses: terraform-linters/setup-tflint@v4
- name: Init TFLint
  run: tflint --init
- name: Run TFLint
  run: tflint --recursive
```

## Output Format Options

```bash
tflint --format=json > tflint-results.json
tflint --format=checkstyle > tflint-checkstyle.xml
```

## Conclusion

TFLint closes the gap between `tofu validate` and real-world errors by catching provider-specific misconfigurations and best practice violations at development time. Integrating it into pre-commit hooks and CI/CD pipelines ensures consistent code quality across your OpenTofu codebase.
