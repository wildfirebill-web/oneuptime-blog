# How to Create API Gateway Stages with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, API Gateway, Infrastructure as Code, DevOps, REST API

Description: Learn how to create and manage AWS API Gateway stages with OpenTofu to deploy REST APIs across development, staging, and production environments.

## Introduction

AWS API Gateway stages allow you to manage multiple deployments of your API, each with independent configuration for logging, throttling, caching, and canary deployments. Using OpenTofu to manage stages ensures consistent configuration across all environments and enables GitOps workflows for API changes.

## Prerequisites

- OpenTofu 1.6+ installed
- AWS credentials configured
- An existing API Gateway REST API or HTTP API

## Project Structure

```text
api-gateway/
├── main.tf
├── variables.tf
├── outputs.tf
└── stages.tf
```

## Defining the REST API and Deployment

Before creating a stage, you need a REST API and a deployment resource. A deployment is a snapshot of your API configuration.

```hcl
# main.tf - REST API definition

resource "aws_api_gateway_rest_api" "example" {
  name        = var.api_name
  description = "Example REST API managed by OpenTofu"

  endpoint_configuration {
    types = ["REGIONAL"]
  }
}

# A deployment is required before creating a stage
resource "aws_api_gateway_deployment" "example" {
  rest_api_id = aws_api_gateway_rest_api.example.id

  # Trigger redeployment when any resource or method changes
  triggers = {
    redeployment = sha1(jsonencode([
      aws_api_gateway_resource.example.id,
      aws_api_gateway_method.example.id,
      aws_api_gateway_integration.example.id,
    ]))
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

## Creating API Gateway Stages

Each environment (dev, staging, prod) gets its own stage with tailored settings.

```hcl
# stages.tf - Define stages for each environment
resource "aws_api_gateway_stage" "prod" {
  deployment_id = aws_api_gateway_deployment.example.id
  rest_api_id   = aws_api_gateway_rest_api.example.id
  stage_name    = "prod"

  # Enable access logging
  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_logs.arn
  }

  # Enable X-Ray tracing for production
  xray_tracing_enabled = true

  # Stage-level throttling defaults
  default_route_settings {
    # Not applicable for REST APIs; use method_settings below
  }

  tags = {
    Environment = "production"
    ManagedBy   = "opentofu"
  }
}

resource "aws_api_gateway_stage" "dev" {
  deployment_id = aws_api_gateway_deployment.example.id
  rest_api_id   = aws_api_gateway_rest_api.example.id
  stage_name    = "dev"

  xray_tracing_enabled = false

  tags = {
    Environment = "development"
    ManagedBy   = "opentofu"
  }
}
```

## Configuring Method-Level Settings

Fine-tune throttling and logging at the method level using `aws_api_gateway_method_settings`.

```hcl
# Apply settings to all methods in the prod stage
resource "aws_api_gateway_method_settings" "prod_all" {
  rest_api_id = aws_api_gateway_rest_api.example.id
  stage_name  = aws_api_gateway_stage.prod.stage_name
  method_path = "*/*"  # applies to all resources and methods

  settings {
    metrics_enabled        = true
    logging_level          = "INFO"
    data_trace_enabled     = false  # disable for prod (verbose)
    throttling_rate_limit  = 1000   # requests/second
    throttling_burst_limit = 2000   # max concurrent requests
    caching_enabled        = true
    cache_ttl_in_seconds   = 300
  }
}
```

## Canary Deployments

OpenTofu supports canary release configurations to gradually shift traffic to new deployments.

```hcl
resource "aws_api_gateway_stage" "prod_canary" {
  rest_api_id   = aws_api_gateway_rest_api.example.id
  deployment_id = aws_api_gateway_deployment.example.id
  stage_name    = "prod"

  canary_settings {
    percent_traffic          = 10    # send 10% of traffic to canary
    deployment_id            = aws_api_gateway_deployment.canary.id
    use_stage_cache          = false
  }
}
```

## Variables and Outputs

```hcl
# variables.tf
variable "api_name" {
  type        = string
  description = "Name of the API Gateway REST API"
  default     = "my-api"
}

# outputs.tf
output "prod_invoke_url" {
  description = "Invoke URL for the production stage"
  value       = aws_api_gateway_stage.prod.invoke_url
}

output "dev_invoke_url" {
  description = "Invoke URL for the development stage"
  value       = aws_api_gateway_stage.dev.invoke_url
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

With OpenTofu you can consistently manage API Gateway stages across environments, configure per-method throttling and caching, and safely roll out changes using canary deployments. Version-controlling your stage configuration enables reliable, auditable API lifecycle management.
