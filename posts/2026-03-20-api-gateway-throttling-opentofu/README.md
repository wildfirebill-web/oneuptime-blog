# How to Configure API Gateway Throttling with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, API Gateway, Throttling, Rate Limiting, Usage Plans, Infrastructure as Code

Description: Learn how to configure API Gateway throttling with OpenTofu to protect backends from traffic spikes and enforce rate limits per client using usage plans and API keys.

## Introduction

API Gateway throttling protects backend services from being overwhelmed by limiting request rates. There are three levels of throttling: account-level defaults (10,000 RPS, 5,000 burst), stage-level settings, and per-method settings. Usage Plans with API Keys enable per-client rate limiting and quota enforcement. When throttled, API Gateway returns HTTP 429 Too Many Requests with a `Retry-After` header.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with API Gateway permissions

## Step 1: Stage-Level Throttling

```hcl
resource "aws_api_gateway_stage" "prod" {
  deployment_id = var.deployment_id
  rest_api_id   = var.rest_api_id
  stage_name    = "prod"
}

resource "aws_api_gateway_method_settings" "throttle" {
  rest_api_id = var.rest_api_id
  stage_name  = aws_api_gateway_stage.prod.stage_name
  method_path = "*/*"  # Apply to all methods

  settings {
    throttling_rate_limit  = 1000   # Steady-state RPS
    throttling_burst_limit = 500    # Max concurrent requests
  }
}

# Per-method throttling (more restrictive than stage level)

resource "aws_api_gateway_method_settings" "expensive_endpoint" {
  rest_api_id = var.rest_api_id
  stage_name  = aws_api_gateway_stage.prod.stage_name
  method_path = "reports/GET"  # resource_path/HTTP_METHOD

  settings {
    throttling_rate_limit  = 10   # 10 RPS for expensive report endpoint
    throttling_burst_limit = 5
  }
}
```

## Step 2: Usage Plans with API Keys

```hcl
resource "aws_api_gateway_usage_plan" "standard" {
  name        = "${var.project_name}-standard-plan"
  description = "Standard API usage plan"

  api_stages {
    api_id = var.rest_api_id
    stage  = aws_api_gateway_stage.prod.stage_name
  }

  throttle_settings {
    rate_limit  = 100   # 100 RPS per key
    burst_limit = 50
  }

  quota_settings {
    limit  = 10000  # 10K requests per day
    period = "DAY"
  }
}

resource "aws_api_gateway_api_key" "client" {
  name    = "${var.project_name}-client-key"
  enabled = true
}

resource "aws_api_gateway_usage_plan_key" "client" {
  key_id        = aws_api_gateway_api_key.client.id
  key_type      = "API_KEY"
  usage_plan_id = aws_api_gateway_usage_plan.standard.id
}

output "api_key_value" {
  value     = aws_api_gateway_api_key.client.value
  sensitive = true
}
```

## Step 3: Deploy

```bash
tofu init
tofu plan
tofu apply

# Test throttling behavior
for i in {1..20}; do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -H "x-api-key: <api-key>" \
    https://<api-id>.execute-api.<region>.amazonaws.com/prod/resource
done

# Check usage plan quotas
aws apigateway get-usage \
  --usage-plan-id <plan-id> \
  --start-date 2026-03-01 \
  --end-date 2026-03-31
```

## Conclusion

Stage-level throttling uses a token bucket algorithm where `rate_limit` is the refill rate and `burst_limit` is the bucket size-clients can burst above the rate limit until tokens are exhausted. Usage Plans require API key authentication (`authorization = "API_KEY"` on methods); any client with a valid key is subject to its plan's limits. Monitor `5XXError` and `ThrottledRequests` CloudWatch metrics to detect when clients are being throttled more than expected and adjust limits accordingly.
