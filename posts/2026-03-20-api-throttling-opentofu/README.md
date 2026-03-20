# How to Configure API Throttling with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Azure, GCP, API Gateway, Rate Limiting, Throttling, Infrastructure as Code

Description: Learn how to configure API throttling and rate limiting across AWS API Gateway, Azure API Management, and GCP API Gateway using OpenTofu to protect backends from traffic spikes.

API throttling prevents abuse, protects backend services from traffic spikes, and ensures fair usage across clients. Each cloud provider has distinct mechanisms — AWS API Gateway uses usage plans and stage-level throttling, Azure APIM uses rate-limit policies, and GCP API Gateway uses quota configurations. Managing throttling in OpenTofu ensures limits are consistently applied and documented.

## AWS API Gateway Throttling

### Stage-Level Default Throttling

```hcl
# HTTP API stage throttling (applies to all routes)
resource "aws_apigatewayv2_stage" "production" {
  api_id      = aws_apigatewayv2_api.main.id
  name        = "production"
  auto_deploy = true

  default_route_settings {
    throttling_burst_limit = 5000   # Max concurrent requests
    throttling_rate_limit  = 10000  # Sustained requests per second
    detailed_metrics_enabled = true
  }
}

# REST API stage throttling
resource "aws_api_gateway_stage" "production" {
  rest_api_id   = aws_api_gateway_rest_api.main.id
  deployment_id = aws_api_gateway_deployment.main.id
  stage_name    = "production"
}

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

### Per-Route Throttling (REST API)

```hcl
resource "aws_api_gateway_method_settings" "search" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  stage_name  = aws_api_gateway_stage.production.stage_name
  method_path = "search/GET"  # Format: {resource_path}/{HTTP_METHOD}

  settings {
    throttling_burst_limit = 100   # Lower limit for expensive search endpoint
    throttling_rate_limit  = 50
    caching_enabled        = true
    cache_ttl_in_seconds   = 300
  }
}
```

### Usage Plans for Per-Client Throttling

```hcl
resource "aws_api_gateway_usage_plan" "free_tier" {
  name = "free-tier"

  api_stages {
    api_id = aws_api_gateway_rest_api.main.id
    stage  = aws_api_gateway_stage.production.stage_name

    throttle {
      path        = "/search/GET"
      burst_limit = 20
      rate_limit  = 10
    }
  }

  throttle_settings {
    burst_limit = 50
    rate_limit  = 25
  }

  quota_settings {
    limit  = 1000  # 1000 requests per day
    period = "DAY"
  }
}

resource "aws_api_gateway_usage_plan" "pro_tier" {
  name = "pro-tier"

  api_stages {
    api_id = aws_api_gateway_rest_api.main.id
    stage  = aws_api_gateway_stage.production.stage_name
  }

  throttle_settings {
    burst_limit = 1000
    rate_limit  = 500
  }

  quota_settings {
    limit  = 100000
    period = "MONTH"
  }
}
```

## AWS WAF Rate-Based Rule (Layer 7 Throttling)

```hcl
# WAF rate limiting at CloudFront/ALB level (before API Gateway)
resource "aws_wafv2_web_acl" "api" {
  name  = "api-rate-limiting"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "api-rate-limit"
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000  # 2000 requests per 5-minute window per IP
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "APIRateLimit"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "APIRateLimiting"
    sampled_requests_enabled   = true
  }
}
```

## Azure API Management Rate Limiting Policy

```hcl
resource "azurerm_api_management_api_policy" "rate_limited" {
  api_name            = azurerm_api_management_api.main.name
  api_management_name = azurerm_api_management.main.name
  resource_group_name = azurerm_resource_group.apim.name

  xml_content = <<-XML
    <policies>
      <inbound>
        <!-- Rate limit by subscription key: 100 calls per 60 seconds -->
        <rate-limit calls="100" renewal-period="60">
          <api name="${azurerm_api_management_api.main.name}" />
        </rate-limit>

        <!-- Monthly quota per subscription -->
        <quota calls="50000" renewal-period="2592000" />

        <!-- IP-based rate limiting for unauthenticated endpoints -->
        <rate-limit-by-key calls="30"
                           renewal-period="60"
                           counter-key="@(context.Request.IpAddress)"
                           increment-condition="@(context.Response.StatusCode == 200)"
                           remaining-calls-header-name="X-Rate-Limit-Remaining"
                           retry-after-header-name="Retry-After" />
        <base />
      </inbound>
      <backend>
        <base />
      </backend>
      <outbound>
        <base />
      </outbound>
    </policies>
  XML
}
```

## CloudWatch Alarms for Throttling

```hcl
resource "aws_cloudwatch_metric_alarm" "api_throttled" {
  alarm_name          = "api-gateway-throttled-requests"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "4XXError"
  namespace           = "AWS/ApiGateway"
  period              = 60
  statistic           = "Sum"
  threshold           = 100

  dimensions = {
    ApiId = aws_apigatewayv2_api.main.id
    Stage = aws_apigatewayv2_stage.production.name
  }

  alarm_description = "High rate of 4xx errors — possible throttling or abuse"
  alarm_actions     = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "throttle_count" {
  alarm_name          = "api-gateway-throttle-count"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Count"  # Use explicit Count for REST APIs
  namespace           = "AWS/ApiGateway"
  period              = 300
  statistic           = "Sum"
  threshold           = 1000

  alarm_description = "API Gateway request volume spiking"
  alarm_actions     = [aws_sns_topic.alerts.arn]
}
```

## Conclusion

API throttling in OpenTofu protects backend services and enforces fair usage policies across clients. In AWS, combine stage-level defaults with per-route overrides for expensive operations, and use usage plans to enforce per-client quotas. Add WAF rate-based rules for IP-level throttling before requests reach API Gateway. In Azure APIM, use rate-limit and quota policies for subscription-based enforcement and rate-limit-by-key for IP-based throttling. Always create CloudWatch alarms on throttling metrics to distinguish legitimate traffic spikes from abuse.
