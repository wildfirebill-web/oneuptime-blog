# How to Use the lower and upper Functions in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the lower and upper functions in OpenTofu to normalize string casing for consistent resource naming and tagging.

## Introduction

The `lower` and `upper` functions in OpenTofu convert strings to all lowercase or all uppercase respectively. They are essential for normalizing user input, creating consistent resource names, and ensuring tags and identifiers follow required casing conventions.

## Syntax

```hcl
lower(string)
upper(string)
```

Both functions take a single string argument and return the case-converted result.

## Basic Examples

```hcl
output "lowercase_example" {
  value = lower("Hello World!")  # Returns "hello world!"
}

output "uppercase_example" {
  value = upper("hello world!")  # Returns "HELLO WORLD!"
}

output "mixed_input_lower" {
  value = lower("PRODUCTION")  # Returns "production"
}

output "mixed_input_upper" {
  value = upper("dev")  # Returns "DEV"
}
```

## Practical Use Cases

### Normalizing Environment Names

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment (case-insensitive)"
  default     = "PRODUCTION"
}

locals {
  # Always use lowercase for resource naming consistency
  env = lower(var.environment)
}

resource "aws_s3_bucket" "app_data" {
  # S3 bucket names must be lowercase
  bucket = "myapp-${local.env}-data"

  tags = {
    Environment = local.env
  }
}
```

### Ensuring Lowercase S3 Bucket Names

AWS S3 bucket names must be lowercase. Use `lower` defensively:

```hcl
variable "project_name" {
  type    = string
  default = "MyProject"
}

resource "aws_s3_bucket" "assets" {
  # Convert to lowercase to satisfy S3 naming requirements
  bucket = lower("${var.project_name}-assets-${var.environment}")
}
```

### Uppercase for Constant-Style Tags

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"

  tags = {
    Name    = "web-server"
    # Use uppercase for environment identifier in tags
    ENV     = upper(var.environment)
    REGION  = upper(replace(var.region, "-", "_"))
  }
}
```

### Case-Insensitive Comparisons

```hcl
variable "deploy_env" {
  type    = string
  default = "Production"
}

locals {
  is_production = lower(var.deploy_env) == "production"
  instance_type = local.is_production ? "m5.large" : "t3.medium"
}
```

### Generating Resource Names

```hcl
variable "team_name" {
  type    = string
  default = "DataEngineering"
}

variable "service_name" {
  type    = string
  default = "Pipeline"
}

locals {
  # Standardize resource names to lowercase with hyphens
  resource_prefix = lower("${var.team_name}-${var.service_name}")
}

resource "aws_ecs_cluster" "main" {
  name = local.resource_prefix  # "dataengineering-pipeline"
}
```

## Step-by-Step Usage

1. Identify string inputs that may have inconsistent casing.
2. Apply `lower()` for identifiers, names, and comparison values.
3. Apply `upper()` for constants, labels, and display values.
4. Test in `tofu console`:

```bash
tofu console

> lower("HELLO")
"hello"
> upper("world")
"WORLD"
> lower("Mixed Case Input")
"mixed case input"
```

## Combining with Other String Functions

```hcl
locals {
  raw_name    = "  My App Name  "
  clean_name  = lower(trimspace(local.raw_name))  # "my app name"
  slug_name   = replace(local.clean_name, " ", "-")  # "my-app-name"
}
```

## Best Practices

- Always use `lower` when creating S3 bucket names, DNS hostnames, and Kubernetes resource names.
- Apply `lower` before string comparisons to make comparisons case-insensitive.
- Use `upper` for environment-like tags where constants are conventional (e.g., `PROD`, `DEV`).

## Conclusion

The `lower` and `upper` functions are foundational string utilities in OpenTofu. By consistently normalizing case, you avoid subtle bugs from case mismatches in resource names, comparisons, and tag values. Make `lower` your default for resource naming and case-insensitive comparisons throughout your infrastructure code.
