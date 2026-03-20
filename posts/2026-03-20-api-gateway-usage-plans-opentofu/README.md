# How to Configure API Gateway Usage Plans with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, API Gateway, Usage Plans, API Keys, Rate Limiting, Infrastructure as Code

Description: Learn how to create API Gateway usage plans and API keys with OpenTofu to throttle and quota API consumers per tier, enabling monetization and fair use enforcement.

## Introduction

API Gateway Usage Plans define throttling limits and quotas for API consumers, identified by API keys. They enable tiered access (free/basic/premium), rate limiting per consumer, and usage metering for billing. Each consumer receives a unique API key associated with a usage plan matching their subscription tier.

## Prerequisites

- OpenTofu v1.6+
- An existing API Gateway REST API with a deployed stage
- AWS credentials with API Gateway permissions

## Step 1: Create Usage Plans for Different Tiers

```hcl
# Free tier usage plan

resource "aws_api_gateway_usage_plan" "free" {
  name        = "${var.project_name}-free-tier"
  description = "Free tier: 100 requests/day, 10 req/s"

  api_stages {
    api_id = var.api_gateway_id
    stage  = "prod"
  }

  throttle_settings {
    burst_limit = 20   # Max concurrent requests
    rate_limit  = 10   # Requests per second
  }

  quota_settings {
    limit  = 100    # Total requests allowed
    period = "DAY"  # Per day quota
  }

  tags = {
    Name = "${var.project_name}-free-tier"
    Tier = "free"
  }
}

# Basic tier usage plan
resource "aws_api_gateway_usage_plan" "basic" {
  name        = "${var.project_name}-basic-tier"
  description = "Basic tier: 10,000 requests/day, 100 req/s"

  api_stages {
    api_id = var.api_gateway_id
    stage  = "prod"
  }

  throttle_settings {
    burst_limit = 200
    rate_limit  = 100
  }

  quota_settings {
    limit  = 10000
    period = "DAY"
  }

  tags = {
    Name = "${var.project_name}-basic-tier"
    Tier = "basic"
  }
}

# Premium tier usage plan
resource "aws_api_gateway_usage_plan" "premium" {
  name        = "${var.project_name}-premium-tier"
  description = "Premium tier: unlimited requests, 1000 req/s"

  api_stages {
    api_id = var.api_gateway_id
    stage  = "prod"
  }

  throttle_settings {
    burst_limit = 2000
    rate_limit  = 1000
  }

  # No quota for premium
  tags = {
    Name = "${var.project_name}-premium-tier"
    Tier = "premium"
  }
}
```

## Step 2: Create API Keys for Customers

```hcl
# API key for a specific customer
resource "aws_api_gateway_api_key" "customer_a" {
  name        = "${var.project_name}-customer-a-key"
  description = "API key for Customer A"
  enabled     = true

  tags = {
    Name     = "${var.project_name}-customer-a-key"
    Customer = "customer-a"
  }
}

# Associate customer API key with a usage plan
resource "aws_api_gateway_usage_plan_key" "customer_a" {
  key_id        = aws_api_gateway_api_key.customer_a.id
  key_type      = "API_KEY"
  usage_plan_id = aws_api_gateway_usage_plan.basic.id
}
```

## Step 3: Require API Key on Methods

```hcl
# Require API key on the GET /users method
resource "aws_api_gateway_method" "get_users" {
  rest_api_id      = var.api_gateway_id
  resource_id      = var.users_resource_id
  http_method      = "GET"
  authorization    = "NONE"
  api_key_required = true  # Require valid API key
}
```

## Step 4: Monitor Usage with CloudWatch

```hcl
resource "aws_cloudwatch_metric_alarm" "api_quota_approaching" {
  alarm_name          = "${var.project_name}-api-quota-warning"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Count"
  namespace           = "AWS/ApiGateway"
  period              = 86400  # 1 day
  statistic           = "Sum"
  threshold           = 8000   # Alert at 80% of basic tier daily quota

  dimensions = {
    ApiName = var.api_name
    Stage   = "prod"
  }

  alarm_actions = [var.sns_topic_arn]
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# Get the API key value (sensitive)
aws apigateway get-api-key \
  --api-key <key-id> \
  --include-value \
  --query 'value' --output text
```

## Conclusion

API Gateway Usage Plans enable per-consumer throttling for multi-tenant APIs, protecting backend services from abuse while enforcing subscription-based quotas. Customers send their API key in the `x-api-key` header, and API Gateway enforces the associated usage plan limits before the request reaches your backend. Combine usage plans with Lambda authorizers for authentication plus key-based usage tracking.
