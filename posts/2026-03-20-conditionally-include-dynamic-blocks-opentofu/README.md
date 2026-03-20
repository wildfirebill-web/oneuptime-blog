# How to Conditionally Include Dynamic Blocks in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Dynamic Blocks, Conditional, HCL, Advanced

Description: Learn how to conditionally include or exclude dynamic blocks in OpenTofu based on variable values, enabling optional resource configuration sections.

## Introduction

Sometimes a resource block should only include certain sub-blocks under specific conditions - for example, only add an HTTPS listener if a certificate is provided, or only include a lifecycle policy if retention is enabled. OpenTofu's dynamic block with a conditional `for_each` achieves this cleanly.

## The Core Pattern: Empty List vs Single-Element List

The key trick is using a conditional expression to return either an empty list (block omitted) or a single-element list (block included).

```hcl
variable "enable_encryption" {
  type    = bool
  default = false
}

variable "kms_key_id" {
  type    = string
  default = ""
}

resource "aws_s3_bucket_server_side_encryption_configuration" "bucket" {
  bucket = aws_s3_bucket.main.id

  rule {
    # Include the CMK block only if encryption is enabled and a key is provided
    dynamic "apply_server_side_encryption_by_default" {
      for_each = var.enable_encryption ? [1] : []
      content {
        sse_algorithm     = "aws:kms"
        kms_master_key_id = var.kms_key_id
      }
    }

    # Default encryption applies when custom encryption is not configured
    dynamic "apply_server_side_encryption_by_default" {
      for_each = var.enable_encryption ? [] : [1]
      content {
        sse_algorithm = "AES256"
      }
    }
  }
}
```

## Conditionally Including HTTPS Listener

```hcl
variable "certificate_arn" {
  description = "ACM certificate ARN for HTTPS. Leave empty to skip HTTPS listener."
  type        = string
  default     = ""
}

resource "aws_lb" "app" {
  name               = "app-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = var.public_subnet_ids
}

# HTTP listener always created

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.app.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    # If HTTPS is configured, redirect HTTP to HTTPS; otherwise forward
    type = var.certificate_arn != "" ? "redirect" : "forward"

    dynamic "redirect" {
      for_each = var.certificate_arn != "" ? [1] : []
      content {
        port        = "443"
        protocol    = "HTTPS"
        status_code = "HTTP_301"
      }
    }

    dynamic "forward" {
      for_each = var.certificate_arn == "" ? [1] : []
      content {
        target_group {
          arn = aws_lb_target_group.app.arn
        }
      }
    }
  }
}
```

## Optional Lifecycle Block

```hcl
variable "prevent_destroy" {
  description = "Set lifecycle prevent_destroy on the S3 bucket"
  type        = bool
  default     = false
}

resource "aws_s3_bucket" "main" {
  bucket = var.bucket_name

  # NOTE: lifecycle blocks cannot use dynamic blocks directly in OpenTofu
  # Use a separate resource or use the ignore_changes workaround instead
}

# The standard approach for optional lifecycle policies is to use separate
# resources conditional on a variable
resource "aws_s3_bucket_lifecycle_configuration" "main" {
  count  = var.enable_lifecycle ? 1 : 0
  bucket = aws_s3_bucket.main.id

  rule {
    id     = "expire-old-objects"
    status = "Enabled"
    expiration {
      days = var.expiration_days
    }
  }
}
```

## Conditionally Including VPC Configuration for Lambda

```hcl
variable "enable_vpc" {
  description = "Deploy the Lambda inside a VPC"
  type        = bool
  default     = false
}

resource "aws_lambda_function" "app" {
  function_name = var.function_name
  role          = aws_iam_role.lambda.arn
  handler       = "index.handler"
  runtime       = "nodejs20.x"
  filename      = var.deployment_package

  # Include VPC config only when enabled
  dynamic "vpc_config" {
    for_each = var.enable_vpc ? [1] : []
    content {
      subnet_ids         = var.vpc_subnet_ids
      security_group_ids = var.vpc_security_group_ids
    }
  }

  # Include tracing config only when X-Ray is enabled
  dynamic "tracing_config" {
    for_each = var.enable_xray ? [1] : []
    content {
      mode = "Active"
    }
  }
}
```

## Conclusion

Conditional dynamic blocks using the `for_each = condition ? [1] : []` pattern give you precise control over which resource sub-blocks are included. This eliminates the need for duplicating resource definitions or using `count` at the resource level just to handle optional configuration sections.
