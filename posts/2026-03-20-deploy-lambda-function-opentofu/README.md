# How to Deploy a Lambda Function with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Lambda, Serverless, Infrastructure as Code

Description: Learn how to deploy an AWS Lambda function with OpenTofu, including IAM roles, environment variables, VPC configuration, and function URL setup.

## Introduction

AWS Lambda lets you run code without provisioning servers. OpenTofu makes it straightforward to deploy Lambda functions with proper IAM permissions, environment variables, VPC networking, and CloudWatch logging.

## Package the Function Code

```hcl
# Archive the function source code into a ZIP file

data "archive_file" "lambda" {
  type        = "zip"
  source_dir  = "${path.module}/src"
  output_path = "${path.module}/dist/function.zip"
}
```

## IAM Execution Role

```hcl
resource "aws_iam_role" "lambda" {
  name = "${var.function_name}-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

# Allow Lambda to write CloudWatch logs
resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# If Lambda runs inside a VPC, it also needs VPC execution permissions
resource "aws_iam_role_policy_attachment" "lambda_vpc" {
  count      = var.enable_vpc ? 1 : 0
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}
```

## Lambda Function

```hcl
resource "aws_lambda_function" "main" {
  function_name = var.function_name
  description   = var.description

  # Use the archived source code
  filename         = data.archive_file.lambda.output_path
  source_code_hash = data.archive_file.lambda.output_base64sha256

  # Runtime and handler
  runtime = var.runtime   # e.g., "nodejs20.x", "python3.12"
  handler = var.handler   # e.g., "index.handler", "lambda_function.lambda_handler"

  role    = aws_iam_role.lambda.arn
  timeout = var.timeout_seconds
  memory_size = var.memory_mb

  environment {
    variables = merge(var.environment_variables, {
      ENVIRONMENT = var.environment
    })
  }

  # Optional: run Lambda inside a VPC
  dynamic "vpc_config" {
    for_each = var.enable_vpc ? [1] : []
    content {
      subnet_ids         = var.private_subnet_ids
      security_group_ids = [aws_security_group.lambda[0].id]
    }
  }

  # Reserved concurrency (set to 0 to throttle completely)
  reserved_concurrent_executions = var.reserved_concurrency

  tags = {
    Name        = var.function_name
    Environment = var.environment
  }

  depends_on = [
    aws_iam_role_policy_attachment.lambda_basic,
    aws_cloudwatch_log_group.lambda,
  ]
}
```

## CloudWatch Log Group

```hcl
# Create the log group before the function to control retention
resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = var.log_retention_days
}
```

## Function URL (Optional)

```hcl
# Expose the Lambda function via a public HTTPS URL
resource "aws_lambda_function_url" "main" {
  function_name      = aws_lambda_function.main.function_name
  authorization_type = "NONE"  # or "AWS_IAM" for authenticated access

  cors {
    allow_credentials = false
    allow_origins     = ["*"]
    allow_methods     = ["GET", "POST"]
    max_age           = 86400
  }
}
```

## EventBridge Schedule Trigger

```hcl
resource "aws_cloudwatch_event_rule" "schedule" {
  name                = "${var.function_name}-schedule"
  schedule_expression = "rate(5 minutes)"
}

resource "aws_cloudwatch_event_target" "lambda" {
  rule      = aws_cloudwatch_event_rule.schedule.name
  target_id = "lambda"
  arn       = aws_lambda_function.main.arn
}

resource "aws_lambda_permission" "eventbridge" {
  statement_id  = "AllowEventBridgeInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.main.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.schedule.arn
}
```

## Outputs

```hcl
output "function_arn"  { value = aws_lambda_function.main.arn }
output "function_name" { value = aws_lambda_function.main.function_name }
output "function_url"  { value = try(aws_lambda_function_url.main.function_url, null) }
```

## Conclusion

Lambda functions deployed with OpenTofu gain consistent IAM permissions, structured logging, and repeatable configuration. Use `source_code_hash` to trigger function updates when code changes, and manage environment variables as first-class OpenTofu inputs.
