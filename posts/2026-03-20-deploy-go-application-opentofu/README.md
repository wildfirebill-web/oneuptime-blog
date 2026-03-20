# How to Deploy a Go Application with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Go, Golang, AWS Lambda, ECS, Infrastructure as Code

Description: Learn how to deploy a Go application on AWS using OpenTofu, covering both serverless Lambda deployment and containerized ECS Fargate options.

## Introduction

Go applications are particularly well-suited for AWS Lambda due to their fast startup times and small binary sizes. This guide covers deploying a Go application as both a Lambda function and as a containerized ECS Fargate service, with the appropriate infrastructure for each.

## Option 1: Lambda Deployment for Go APIs

Go compiles to a single binary, making it ideal for Lambda.

```hcl
# The Go binary is built and zipped before Terraform runs

# Build command: GOOS=linux GOARCH=amd64 go build -o bootstrap main.go && zip function.zip bootstrap

data "archive_file" "go_app" {
  type        = "zip"
  source_file = "${path.module}/build/bootstrap"  # compiled Go binary
  output_path = "${path.module}/build/function.zip"
}

resource "aws_lambda_function" "go_api" {
  function_name    = "myapp-go-api-${var.environment}"
  filename         = data.archive_file.go_app.output_path
  source_code_hash = data.archive_file.go_app.output_base64sha256
  handler          = "bootstrap"  # Go uses 'bootstrap' by convention
  runtime          = "provided.al2023"  # custom runtime for Go
  role             = aws_iam_role.lambda.arn
  timeout          = 30
  memory_size      = 128  # Go is memory-efficient

  environment {
    variables = {
      ENVIRONMENT = var.environment
      LOG_LEVEL   = var.environment == "prod" ? "info" : "debug"
    }
  }

  # Snap start not available for custom runtimes, but Go starts fast anyway
}
```

## API Gateway for Go Lambda

```hcl
resource "aws_apigatewayv2_api" "go_api" {
  name          = "myapp-go-api"
  protocol_type = "HTTP"

  cors_configuration {
    allow_origins = var.allowed_origins
    allow_methods = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
    allow_headers = ["Content-Type", "Authorization"]
  }
}

resource "aws_apigatewayv2_integration" "go_lambda" {
  api_id                 = aws_apigatewayv2_api.go_api.id
  integration_type       = "AWS_PROXY"
  integration_uri        = aws_lambda_function.go_api.invoke_arn
  payload_format_version = "2.0"
}

resource "aws_apigatewayv2_route" "go_proxy" {
  api_id    = aws_apigatewayv2_api.go_api.id
  route_key = "ANY /{proxy+}"
  target    = "integrations/${aws_apigatewayv2_integration.go_lambda.id}"
}

resource "aws_apigatewayv2_stage" "go_api" {
  api_id      = aws_apigatewayv2_api.go_api.id
  name        = var.environment
  auto_deploy = true
}
```

## Option 2: ECS Fargate for Long-Running Go Services

For services needing persistent connections (gRPC, WebSockets, streaming).

```hcl
resource "aws_ecs_task_definition" "go_service" {
  family                   = "myapp-go-service"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"    # Go is CPU-efficient
  memory                   = "512"    # Go is memory-efficient
  execution_role_arn       = aws_iam_role.ecs_execution.arn

  container_definitions = jsonencode([{
    name  = "go-service"
    image = "${aws_ecr_repository.go_app.repository_url}:${var.app_version}"

    environment = [
      { name = "APP_ENV",  value = var.environment },
      { name = "APP_PORT", value = "8080" },
    ]

    secrets = [
      { name = "DATABASE_URL", valueFrom = aws_secretsmanager_secret.db_url.arn },
      { name = "API_KEY",      valueFrom = aws_secretsmanager_secret.api_key.arn },
    ]

    portMappings = [{
      containerPort = 8080
      protocol      = "tcp"
    }]

    healthCheck = {
      # Go apps start instantly, short grace period needed
      command     = ["CMD-SHELL", "wget -q -O- http://localhost:8080/health || exit 1"]
      interval    = 15
      timeout     = 5
      retries     = 3
      startPeriod = 10  # Go starts in milliseconds
    }

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/myapp-go"
        "awslogs-region"        = var.region
        "awslogs-stream-prefix" = "go-service"
      }
    }
  }])
}
```

## Lambda Cost Comparison

```hcl
# CloudWatch alarm for Lambda cost optimization
resource "aws_cloudwatch_metric_alarm" "lambda_duration" {
  alarm_name          = "go-lambda-high-duration"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "Duration"
  namespace           = "AWS/Lambda"
  period              = 300
  statistic           = "Average"
  threshold           = 5000  # 5 seconds average is too slow for Lambda
  alarm_description   = "Consider migrating to ECS if duration stays high"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    FunctionName = aws_lambda_function.go_api.function_name
  }
}
```

## Summary

Go applications are particularly efficient on Lambda due to fast cold starts (typically under 50ms), small binary size, and low memory consumption. Use Lambda + API Gateway for HTTP APIs and microservices, and ECS Fargate for services requiring persistent connections like gRPC servers, WebSocket services, or long-running background workers. The `provided.al2023` runtime uses the custom runtime interface, and Go binaries should be compiled as `bootstrap` for Lambda. Set `startPeriod` to a short value (10s) for Go containers since they start almost instantly.
