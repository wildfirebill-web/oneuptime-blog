# How to Document OpenTofu Modules Properly

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Modules

Description: Learn best practices for documenting OpenTofu modules including README structure, variable and output descriptions, and auto-generated documentation with terraform-docs.

## Introduction

Good documentation transforms a module from a private tool into a reusable asset. Well-documented modules reduce onboarding time, prevent misconfiguration, and make the module registry page informative and trustworthy.

## Variable Descriptions

Every variable should have a clear `description` and appropriate `type`:

```hcl
# variables.tf
variable "name" {
  type        = string
  description = "Name prefix applied to all resources created by this module"
}

variable "environment" {
  type        = string
  description = "Deployment environment. Must be one of: dev, staging, prod"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod"
  }
}

variable "enable_flow_logs" {
  type        = bool
  description = "Whether to enable VPC flow logs. Enables CloudWatch log group creation when true"
  default     = false
}
```

## Output Descriptions

Every output should describe what it contains and how to use it:

```hcl
# outputs.tf
output "vpc_id" {
  description = "The ID of the created VPC. Pass to resources that require vpc_id."
  value       = aws_vpc.this.id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs, one per availability zone. Use for EKS node groups and RDS instances."
  value       = aws_subnet.private[*].id
}
```

## README Structure

A good README contains these sections:

```markdown
# terraform-aws-vpc

Brief description of what this module creates.

## Usage

Minimal working example — paste and run:

```hcl
module "vpc" {
  source  = "your-org/vpc/aws"
  version = "~> 2.0"

  name        = "production"
  cidr_block  = "10.0.0.0/16"
  environment = "prod"
}
```

## Requirements

| Name | Version |
|------|---------|
| opentofu | >= 1.6 |
| aws | >= 5.0 |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| name | Name prefix | string | — | yes |
| cidr_block | VPC CIDR block | string | 10.0.0.0/16 | no |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | The ID of the VPC |
| private_subnet_ids | List of private subnet IDs |
```

## Auto-Generating Documentation with terraform-docs

`terraform-docs` generates the inputs/outputs tables automatically from your variable and output descriptions:

```bash
# Install terraform-docs
brew install terraform-docs

# Generate README section
terraform-docs markdown table . > DOCS.md

# Inject into README (between markers)
terraform-docs markdown table --output-file README.md --output-mode inject .
```

Add markers to your README to control where generated content is inserted:

```markdown
<!-- BEGIN_TF_DOCS -->
<!-- END_TF_DOCS -->
```

## Automate with .terraform-docs.yml

```yaml
# .terraform-docs.yml
formatter: markdown table

output:
  file: README.md
  mode: inject

sections:
  show:
    - inputs
    - outputs
    - requirements
    - providers
```

## CI/CD Documentation Check

```yaml
# .github/workflows/docs.yml
name: Documentation Check
on: [pull_request]
jobs:
  docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: terraform-docs/gh-actions@v1
        with:
          output-file: README.md
          output-method: inject
          fail-on-diff: true
```

## Conclusion

Proper module documentation reduces friction for consumers and increases module adoption. Write clear variable and output descriptions at authoring time, maintain a README with a usage example and requirements table, and automate documentation generation with `terraform-docs` to keep it in sync with your code. Good documentation is the most visible form of module quality.
