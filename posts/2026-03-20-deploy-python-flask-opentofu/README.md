# How to Deploy a Python Flask Application with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Python, Flask, AWS Lambda, API Gateway, Infrastructure as Code

Description: Learn how to deploy a Python Flask application on AWS using OpenTofu, with both serverless Lambda and containerized ECS Fargate deployment options.

## Introduction

Python Flask applications can be deployed serverlessly using AWS Lambda with API Gateway (excellent for low-to-medium traffic) or containerized on ECS Fargate (better for high-traffic, long-running requests). This guide covers both options.

## Option 1: Serverless Deployment with Lambda

Use Mangum to wrap Flask as an ASGI/WSGI handler for Lambda.

```hcl
# Package the Flask app as a ZIP

data "archive_file" "flask_app" {
  type        = "zip"
  source_dir  = "${path.module}/app"
  output_path = "${path.module}/flask_app.zip"
}

resource "aws_lambda_function" "flask_app" {
  function_name    = "myapp-flask-${var.environment}"
  filename         = data.archive_file.flask_app.output_path
  source_code_hash = data.archive_file.flask_app.output_base64sha256
  handler          = "app.handler"  # app.py, handler function (Mangum)
  runtime          = "python3.11"
  role             = aws_iam_role.lambda.arn
  timeout          = 30
  memory_size      = 512

  environment {
    variables = {
      FLASK_ENV           = var.environment
      DATABASE_URL        = "postgresql://..."  # use SSM in production
      SECRET_KEY_ARN      = aws_secretsmanager_secret.flask_key.arn
    }
  }

  layers = [aws_lambda_layer_version.dependencies.arn]
}

resource "aws_lambda_layer_version" "dependencies" {
  filename            = "${path.module}/dependencies.zip"
  layer_name          = "flask-dependencies"
  compatible_runtimes = ["python3.11"]
}
```

## API Gateway for Lambda

```hcl
resource "aws_apigatewayv2_api" "flask" {
  name          = "myapp-flask-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "flask_lambda" {
  api_id                 = aws_apigatewayv2_api.flask.id
  integration_type       = "AWS_PROXY"
  integration_uri        = aws_lambda_function.flask_app.invoke_arn
  payload_format_version = "2.0"
}

resource "aws_apigatewayv2_route" "flask_proxy" {
  api_id    = aws_apigatewayv2_api.flask.id
  route_key = "ANY /{proxy+}"
  target    = "integrations/${aws_apigatewayv2_integration.flask_lambda.id}"
}

resource "aws_apigatewayv2_stage" "flask" {
  api_id      = aws_apigatewayv2_api.flask.id
  name        = var.environment
  auto_deploy = true

  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_gateway.arn
  }
}

resource "aws_lambda_permission" "api_gateway" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.flask_app.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.flask.execution_arn}/*/*"
}
```

## Option 2: Container Deployment on ECS Fargate

For production Flask applications with Gunicorn.

```hcl
resource "aws_ecs_task_definition" "flask" {
  family                   = "myapp-flask"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_execution.arn

  container_definitions = jsonencode([{
    name  = "flask-app"
    image = "${aws_ecr_repository.app.repository_url}:${var.app_version}"
    # Dockerfile CMD: gunicorn --workers 4 --bind 0.0.0.0:8000 app:app

    environment = [
      { name = "FLASK_ENV",     value = var.environment },
      { name = "WORKERS",       value = "4" },
    ]

    secrets = [
      { name = "DATABASE_URL",  valueFrom = aws_secretsmanager_secret.db_url.arn },
      { name = "FLASK_SECRET",  valueFrom = aws_secretsmanager_secret.flask_key.arn },
    ]

    portMappings = [{
      containerPort = 8000
      protocol      = "tcp"
    }]

    healthCheck = {
      command  = ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"]
      interval = 30
      timeout  = 5
      retries  = 3
    }

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/myapp-flask"
        "awslogs-region"        = var.region
        "awslogs-stream-prefix" = "flask"
      }
    }
  }])
}
```

## CloudWatch Alarms for Flask App

```hcl
resource "aws_cloudwatch_metric_alarm" "flask_errors" {
  alarm_name          = "myapp-flask-5xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "5XXError"
  namespace           = "AWS/ApiGateway"
  period              = 60
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "Flask API 5XX errors exceeded threshold"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ApiName = aws_apigatewayv2_api.flask.name
    Stage   = aws_apigatewayv2_stage.flask.name
  }
}
```

## Summary

Flask applications deploy well on both AWS Lambda (via Mangum WSGI wrapper) for low-cost serverless operation, and ECS Fargate with Gunicorn for production-grade performance. Use Lambda for simple APIs with low-to-medium traffic and cost optimization, and ECS for applications requiring consistent performance, long-running requests, or WebSocket support. Both approaches use Secrets Manager for sensitive configuration and CloudWatch for logging and monitoring.
