# How to Create Lambda Functions with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, AWS, Lambda, Terraform, IaC, DevOps, Serverless

Description: Learn how to create and configure AWS Lambda functions with OpenTofu, including deployment packages, environment variables, VPC configuration, and event triggers.

## Introduction

Creating Lambda functions with OpenTofu involves packaging the function code, creating the IAM execution role, configuring the function itself, and optionally setting up triggers. OpenTofu manages both the infrastructure (IAM, VPC security groups, CloudWatch logs) and the function deployment.

## Lambda Execution Role

```hcl
data "aws_iam_policy_document" "lambda_assume_role" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "lambda" {
  name               = "${var.function_name}-role"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume_role.json
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

## Packaging Function Code

```hcl
# Archive the Lambda function code

data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/lambda/handler"
  output_path = "${path.module}/dist/function.zip"
}
```

## Basic Lambda Function

```hcl
resource "aws_lambda_function" "main" {
  function_name    = var.function_name
  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  role    = aws_iam_role.lambda.arn
  handler = "index.handler"
  runtime = "nodejs20.x"

  timeout     = 30
  memory_size = 256

  environment {
    variables = {
      ENVIRONMENT = var.environment
      DB_HOST     = var.db_host
      BUCKET_NAME = var.bucket_name
    }
  }

  tags = {
    Name        = var.function_name
    Environment = var.environment
  }
}
```

## Lambda from S3 (For Large Functions)

```hcl
# Upload to S3 first
resource "aws_s3_object" "lambda_zip" {
  bucket = aws_s3_bucket.lambda_artifacts.bucket
  key    = "${var.function_name}/${data.archive_file.lambda_zip.output_md5}.zip"
  source = data.archive_file.lambda_zip.output_path
  etag   = data.archive_file.lambda_zip.output_md5
}

resource "aws_lambda_function" "main" {
  function_name = var.function_name
  s3_bucket     = aws_s3_object.lambda_zip.bucket
  s3_key        = aws_s3_object.lambda_zip.key
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  role    = aws_iam_role.lambda.arn
  handler = "index.handler"
  runtime = "nodejs20.x"
}
```

## Lambda in VPC

```hcl
resource "aws_iam_role_policy_attachment" "lambda_vpc" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

resource "aws_security_group" "lambda" {
  name   = "${var.function_name}-sg"
  vpc_id = var.vpc_id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_lambda_function" "vpc_lambda" {
  function_name = var.function_name
  filename      = data.archive_file.lambda_zip.output_path
  role          = aws_iam_role.lambda.arn
  handler       = "index.handler"
  runtime       = "nodejs20.x"

  vpc_config {
    subnet_ids         = var.private_subnet_ids
    security_group_ids = [aws_security_group.lambda.id]
  }
}
```

## CloudWatch Log Group

```hcl
resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = 30
}
```

## API Gateway Trigger

```hcl
resource "aws_lambda_permission" "api_gateway" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.main.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.main.execution_arn}/*/*"
}
```

## S3 Event Trigger

```hcl
resource "aws_lambda_permission" "s3" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.main.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.trigger.arn
}

resource "aws_s3_bucket_notification" "trigger" {
  bucket = aws_s3_bucket.trigger.id

  lambda_function {
    lambda_function_arn = aws_lambda_function.main.arn
    events              = ["s3:ObjectCreated:*"]
    filter_suffix       = ".json"
  }

  depends_on = [aws_lambda_permission.s3]
}
```

## Outputs

```hcl
output "function_arn"  { value = aws_lambda_function.main.arn }
output "function_name" { value = aws_lambda_function.main.function_name }
output "invoke_arn"    { value = aws_lambda_function.main.invoke_arn }
```

## Conclusion

Creating Lambda functions with OpenTofu involves the execution role, the function deployment, and CloudWatch log group configuration. Use `source_code_hash` to trigger redeployment when code changes. For VPC-hosted Lambda functions, attach the VPC execution policy and configure security groups. Grant invocation permissions to each event source with `aws_lambda_permission`. Consider the `terraform-aws-modules/lambda/aws` module for complex Lambda setups with multiple layers and event sources.
