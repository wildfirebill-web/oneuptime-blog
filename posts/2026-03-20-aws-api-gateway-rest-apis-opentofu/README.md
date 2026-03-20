# How to Create AWS API Gateway REST APIs with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, API Gateway, REST API, Lambda, Infrastructure as Code

Description: Learn how to create AWS API Gateway REST APIs with OpenTofu, including Lambda integration, request validation, usage plans, API keys, and deployment stages.

AWS API Gateway REST APIs provide a fully managed service for creating, publishing, and securing APIs. REST APIs offer the most features including request validation, API keys, usage plans, and caching. Managing REST APIs in OpenTofu ensures consistent API configuration with proper security and throttling controls.

## REST API with Lambda Integration

```hcl
# Create the REST API
resource "aws_api_gateway_rest_api" "main" {
  name        = "${var.service_name}-api"
  description = "REST API for ${var.service_name}"

  endpoint_configuration {
    types = ["REGIONAL"]  # REGIONAL, EDGE, or PRIVATE
  }

  # Request body size limit (10 MB)
  minimum_compression_size = 0

  tags = {
    Environment = var.environment
  }
}

# Create a resource path: /users
resource "aws_api_gateway_resource" "users" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  parent_id   = aws_api_gateway_rest_api.main.root_resource_id
  path_part   = "users"
}

# GET /users method
resource "aws_api_gateway_method" "get_users" {
  rest_api_id      = aws_api_gateway_rest_api.main.id
  resource_id      = aws_api_gateway_resource.users.id
  http_method      = "GET"
  authorization    = "COGNITO_USER_POOLS"
  authorizer_id    = aws_api_gateway_authorizer.cognito.id
  api_key_required = false
}

# Lambda integration for GET /users
resource "aws_api_gateway_integration" "get_users" {
  rest_api_id             = aws_api_gateway_rest_api.main.id
  resource_id             = aws_api_gateway_resource.users.id
  http_method             = aws_api_gateway_method.get_users.http_method
  integration_http_method = "POST"  # Lambda always uses POST
  type                    = "AWS_PROXY"  # Pass full request to Lambda
  uri                     = aws_lambda_function.users.invoke_arn
}
```

## Cognito Authorizer

```hcl
resource "aws_api_gateway_authorizer" "cognito" {
  name          = "cognito-authorizer"
  rest_api_id   = aws_api_gateway_rest_api.main.id
  type          = "COGNITO_USER_POOLS"
  provider_arns = [aws_cognito_user_pool.main.arn]

  identity_source = "method.request.header.Authorization"
}
```

## Request Validation

```hcl
resource "aws_api_gateway_request_validator" "body_and_params" {
  name                        = "validate-body-and-params"
  rest_api_id                 = aws_api_gateway_rest_api.main.id
  validate_request_body       = true
  validate_request_parameters = true
}

# POST /users with body validation
resource "aws_api_gateway_method" "create_user" {
  rest_api_id          = aws_api_gateway_rest_api.main.id
  resource_id          = aws_api_gateway_resource.users.id
  http_method          = "POST"
  authorization        = "COGNITO_USER_POOLS"
  authorizer_id        = aws_api_gateway_authorizer.cognito.id
  request_validator_id = aws_api_gateway_request_validator.body_and_params.id

  # Define expected request model
  request_models = {
    "application/json" = aws_api_gateway_model.create_user.name
  }
}

resource "aws_api_gateway_model" "create_user" {
  rest_api_id  = aws_api_gateway_rest_api.main.id
  name         = "CreateUserRequest"
  content_type = "application/json"

  schema = jsonencode({
    "$schema" = "http://json-schema.org/draft-04/schema#"
    type      = "object"
    required  = ["email", "name"]
    properties = {
      email = { type = "string", format = "email" }
      name  = { type = "string", minLength = 1 }
    }
  })
}
```

## Deployment and Stage

```hcl
resource "aws_api_gateway_deployment" "main" {
  rest_api_id = aws_api_gateway_rest_api.main.id

  # Force new deployment when API changes
  triggers = {
    redeployment = sha1(jsonencode([
      aws_api_gateway_resource.users.id,
      aws_api_gateway_method.get_users.id,
      aws_api_gateway_integration.get_users.id,
    ]))
  }

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_api_gateway_stage" "production" {
  deployment_id = aws_api_gateway_deployment.main.id
  rest_api_id   = aws_api_gateway_rest_api.main.id
  stage_name    = "production"

  # Enable access logging to CloudWatch
  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_access.arn
    format = jsonencode({
      requestId      = "$context.requestId"
      ip             = "$context.identity.sourceIp"
      requestTime    = "$context.requestTime"
      httpMethod     = "$context.httpMethod"
      routeKey       = "$context.routeKey"
      status         = "$context.status"
      responseLength = "$context.responseLength"
      latency        = "$context.integrationLatency"
    })
  }

  # Enable caching (reduces Lambda invocations)
  cache_cluster_enabled = true
  cache_cluster_size    = "0.5"  # 0.5 GB cache

  xray_tracing_enabled = true  # Enable X-Ray tracing
}

# Stage-level throttling for REST APIs uses aws_api_gateway_method_settings
resource "aws_api_gateway_method_settings" "all" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  stage_name  = aws_api_gateway_stage.production.stage_name
  method_path = "*/*"  # Apply to all methods

  settings {
    throttling_burst_limit = 5000
    throttling_rate_limit  = 10000
  }
}
```

## Usage Plans and API Keys

```hcl
resource "aws_api_gateway_api_key" "mobile_app" {
  name = "mobile-app-key"
}

resource "aws_api_gateway_usage_plan" "basic" {
  name = "basic-plan"

  api_stages {
    api_id = aws_api_gateway_rest_api.main.id
    stage  = aws_api_gateway_stage.production.stage_name
  }

  throttle_settings {
    burst_limit = 100   # Max concurrent requests
    rate_limit  = 50    # Requests per second
  }

  quota_settings {
    limit  = 10000  # Max requests per period
    period = "MONTH"
  }
}

resource "aws_api_gateway_usage_plan_key" "mobile_app" {
  key_id        = aws_api_gateway_api_key.mobile_app.id
  key_type      = "API_KEY"
  usage_plan_id = aws_api_gateway_usage_plan.basic.id
}
```

## Conclusion

AWS API Gateway REST APIs in OpenTofu provide fully managed API hosting with built-in authentication, request validation, throttling, and caching. Use Cognito authorizers for JWT-based authentication, request models with JSON Schema for input validation, usage plans and API keys for rate limiting per-client, and access logging to CloudWatch for troubleshooting. Always set create_before_destroy on deployments to avoid downtime during API updates.
