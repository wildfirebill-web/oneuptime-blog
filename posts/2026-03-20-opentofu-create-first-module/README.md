# How to Create Your First OpenTofu Module

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Modules, Best Practice

Description: Learn how to create your first reusable OpenTofu module to encapsulate infrastructure patterns and share them across projects.

## What is an OpenTofu Module?

A module is a collection of `.tf` files in a directory that represent a reusable infrastructure component. Every OpenTofu configuration is technically a module (the "root module"), but you can also create child modules that are called from parent configurations.

Modules let you:
- Encapsulate complex infrastructure into a simple interface
- Reuse the same pattern across multiple environments
- Share infrastructure patterns across teams

## Module Structure

A minimal module requires three files:

```text
modules/
└── s3-website/
    ├── main.tf        # Resource definitions
    ├── variables.tf   # Input variables
    └── outputs.tf     # Output values
```

## Step 1: Create variables.tf

Define the inputs your module accepts.

```hcl
# modules/s3-website/variables.tf

variable "bucket_name" {
  type        = string
  description = "The name of the S3 bucket to create"
}

variable "environment" {
  type        = string
  description = "Environment name (dev, staging, production)"
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "tags" {
  type        = map(string)
  description = "Additional tags to apply to all resources"
  default     = {}
}
```

## Step 2: Create main.tf

Define the resources the module creates.

```hcl
# modules/s3-website/main.tf

resource "aws_s3_bucket" "website" {
  bucket = var.bucket_name

  tags = merge(var.tags, {
    Name        = var.bucket_name
    Environment = var.environment
    ManagedBy   = "OpenTofu"
  })
}

resource "aws_s3_bucket_website_configuration" "website" {
  bucket = aws_s3_bucket.website.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}

resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}
```

## Step 3: Create outputs.tf

Expose values that callers may need.

```hcl
# modules/s3-website/outputs.tf

output "bucket_id" {
  description = "The name/ID of the S3 bucket"
  value       = aws_s3_bucket.website.id
}

output "bucket_arn" {
  description = "The ARN of the S3 bucket"
  value       = aws_s3_bucket.website.arn
}

output "website_endpoint" {
  description = "The S3 website endpoint URL"
  value       = aws_s3_bucket_website_configuration.website.website_endpoint
}
```

## Step 4: Call the Module

Use the module from your root configuration.

```hcl
# main.tf (root module)

module "dev_website" {
  source = "./modules/s3-website"

  bucket_name = "my-app-dev-website"
  environment = "dev"
  tags = {
    Project = "my-app"
    Team    = "frontend"
  }
}

module "prod_website" {
  source = "./modules/s3-website"

  bucket_name = "my-app-production-website"
  environment = "production"
  tags = {
    Project = "my-app"
    Team    = "frontend"
  }
}

# Access module outputs

output "dev_website_url" {
  value = module.dev_website.website_endpoint
}
```

## Initialize and Apply

```bash
# Initialize to download providers and resolve module sources
tofu init

# Preview what will be created
tofu plan

# Create the resources
tofu apply
```

## Important Notes

- Each module should solve one well-defined problem. Avoid creating "god modules" that do everything.
- Variables should have clear descriptions and validation rules where appropriate.
- Outputs should expose everything a caller might reasonably need.
- Run `tofu fmt` in the module directory to ensure consistent formatting.

## Conclusion

Creating your first OpenTofu module is straightforward: define inputs in `variables.tf`, resources in `main.tf`, and outputs in `outputs.tf`. Then call the module with a `module` block. Modules are the foundation of reusable, maintainable infrastructure as code.
