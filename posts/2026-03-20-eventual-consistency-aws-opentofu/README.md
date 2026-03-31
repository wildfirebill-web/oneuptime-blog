# How to Handle Eventual Consistency with AWS Resources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Eventual Consistency, AWS, IAM, Best Practice, Infrastructure as Code

Description: Learn how to handle AWS eventual consistency issues in OpenTofu where resources appear created but aren't immediately available for subsequent operations.

## Introduction

AWS is an eventually consistent system. When you create an IAM role, it may take seconds to propagate globally. When you create a security group, it may not be immediately visible to another API call. OpenTofu may receive success from the creation API but then fail on subsequent operations that depend on the resource being fully available.

## IAM Propagation Delay

IAM changes can take 5-10 seconds to propagate globally.

```hcl
resource "aws_iam_role" "lambda" {
  name = "${var.app_name}-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# Add explicit sleep to allow IAM to propagate before Lambda creation

resource "time_sleep" "iam_propagation" {
  depends_on = [aws_iam_role_policy_attachment.lambda_basic]

  create_duration = "10s"
}

resource "aws_lambda_function" "app" {
  depends_on    = [time_sleep.iam_propagation]
  function_name = "${var.app_name}-function"
  role          = aws_iam_role.lambda.arn
  # ...
}
```

## S3 Bucket Policy Consistency

S3 bucket policies have eventual consistency. Applying a bucket policy and immediately reading from the bucket can fail.

```hcl
resource "aws_s3_bucket_policy" "main" {
  bucket = aws_s3_bucket.main.id
  policy = jsonencode({ ... })
}

# Wait for policy to take effect before any Lambda or other resources
# that need to read from the bucket
resource "time_sleep" "bucket_policy" {
  depends_on = [aws_s3_bucket_policy.main]
  create_duration = "5s"
}
```

## Using the time Provider

```hcl
terraform {
  required_providers {
    time = {
      source  = "hashicorp/time"
      version = "~> 0.9"
    }
  }
}
```

## Retry Logic in null_resource

For critical operations, add polling until the resource is actually available.

```hcl
resource "null_resource" "verify_iam_ready" {
  depends_on = [aws_iam_role.app]

  triggers = {
    role_arn = aws_iam_role.app.arn
  }

  provisioner "local-exec" {
    command = <<-SCRIPT
      # Verify the role is accessible before proceeding
      MAX_ATTEMPTS=12
      ATTEMPT=0
      until aws iam get-role --role-name "${aws_iam_role.app.name}" --query 'Role.Arn' --output text 2>/dev/null; do
        ATTEMPT=$((ATTEMPT + 1))
        if [[ $ATTEMPT -ge $MAX_ATTEMPTS ]]; then
          echo "IAM role not available after $MAX_ATTEMPTS attempts"
          exit 1
        fi
        echo "Waiting for IAM role to be available (attempt $ATTEMPT)..."
        sleep 5
      done
      echo "IAM role confirmed available"
    SCRIPT
  }
}
```

## CloudFront and DNS Propagation

```hcl
# CloudFront distributions take 5-15 minutes to deploy globally
resource "aws_cloudfront_distribution" "main" {
  # ...
}

# If downstream resources need the CloudFront domain immediately,
# add a wait after the distribution becomes enabled
resource "time_sleep" "cloudfront_propagation" {
  depends_on = [aws_cloudfront_distribution.main]
  # CloudFront status changes to "Deployed" before this matters,
  # but DNS propagation for Route53 alias records takes additional time
  create_duration = "30s"
}
```

## Addressing Race Conditions in Parallel Creates

```hcl
# When creating many resources that all need the same IAM role,
# use depends_on to avoid race conditions
locals {
  iam_role_arn = aws_iam_role.shared.arn
}

resource "time_sleep" "shared_iam_ready" {
  depends_on      = [aws_iam_role.shared]
  create_duration = "15s"
}

# All Lambda functions wait for IAM to be ready
resource "aws_lambda_function" "functions" {
  for_each   = var.functions
  depends_on = [time_sleep.shared_iam_ready]
  role       = local.iam_role_arn
  # ...
}
```

## Summary

AWS eventual consistency requires explicit delays after IAM, S3 policy, and DNS changes. The `time_sleep` resource from the HashiCorp time provider adds deterministic waits, while `null_resource` with polling provides adaptive waits for unpredictable propagation times. Always add these waits when resources depend on recently modified IAM, policies, or DNS records.
