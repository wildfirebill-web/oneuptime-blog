# How to Configure Health Checks for AWS ALB IPv4 Target Groups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, ALB, Health Checks, IPv4, Load Balancing, Target Groups

Description: Configure HTTP and HTTPS health checks for AWS Application Load Balancer target groups to ensure only healthy IPv4 targets receive traffic.

## Introduction

Health checks are how an AWS ALB determines whether a target (EC2 instance, IP, or Lambda) is ready to receive traffic. A target must pass health checks before the ALB sends requests to it, and it is removed from rotation if checks start failing. Proper health check configuration is critical for zero-downtime deployments and automatic failover.

## Health Check Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| Protocol | HTTP or HTTPS | HTTP |
| Path | URL path to check | / |
| Port | Port to check on (or traffic-port) | traffic-port |
| Healthy threshold | Consecutive successes needed | 5 |
| Unhealthy threshold | Consecutive failures before removal | 2 |
| Timeout | Response timeout (seconds) | 5 |
| Interval | Check interval (seconds) | 30 |
| Success codes | HTTP status codes considered healthy | 200 |

## Creating a Target Group with Health Checks

```bash
# Create a target group with custom health check settings
aws elbv2 create-target-group \
  --name my-app-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-0123456789abcdef0 \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-path /health \          # Dedicated health check endpoint
  --health-check-port 8080 \
  --healthy-threshold-count 2 \          # Require 2 consecutive successes
  --unhealthy-threshold-count 3 \        # Remove after 3 failures
  --health-check-timeout-seconds 5 \
  --health-check-interval-seconds 15 \   # Check every 15 seconds
  --matcher HttpCode=200-299             # Accept any 2xx as healthy
```

## Modifying Health Checks on an Existing Target Group

```bash
# Update health check settings for an existing target group
aws elbv2 modify-target-group \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-app-tg/abc123 \
  --health-check-path /api/health \
  --health-check-interval-seconds 10 \
  --healthy-threshold-count 2 \
  --matcher HttpCode=200
```

## Checking Target Health Status

```bash
# Describe health of targets in the group
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-app-tg/abc123 \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,Health:TargetHealth.State,Reason:TargetHealth.Reason}'
```

## Implementing a Health Endpoint in Your Application

Your application should expose a dedicated health check endpoint:

```python
# Example Flask health check endpoint
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/health')
def health_check():
    # Check database connectivity
    try:
        db.execute("SELECT 1")
        return jsonify({"status": "healthy"}), 200
    except Exception:
        return jsonify({"status": "unhealthy"}), 503
```

## Terraform Configuration

```hcl
resource "aws_lb_target_group" "app" {
  name     = "my-app-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    path                = "/health"
    protocol            = "HTTP"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 15
    matcher             = "200-299"
  }
}
```

## Common Health Check Mistakes

- **Too aggressive thresholds**: Short intervals with low thresholds cause flapping during deployments
- **Checking `/` instead of `/health`**: Heavy homepage responses slow health checks and inflate metrics
- **Ignoring 503 on shutdown**: Configure your app to return 503 on `/health` during graceful shutdown to drain connections cleanly

## Conclusion

Properly tuned health checks are fundamental to reliable load balancing. Use a lightweight dedicated endpoint, set conservative thresholds for production, and verify target health after every deployment.
