# How to Configure ELB Target Groups with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, ELB, Target Groups, Load Balancing, Health Checks, Infrastructure as Code

Description: Learn how to configure Elastic Load Balancing target groups with OpenTofu for ALB, NLB, and GWLB, including health checks, stickiness, and target registration.

## Introduction

ELB target groups define where load balancers route requests and how they check target health. Target groups support EC2 instances, IP addresses, Lambda functions, or other ALBs as targets. Each target group has its own health check configuration, deregistration delay, and load balancing algorithm. Understanding target group configuration is fundamental to building resilient load-balanced applications on AWS.

## Prerequisites

- OpenTofu v1.6+
- An Application Load Balancer, Network Load Balancer, or Gateway Load Balancer
- AWS credentials with EC2 and ELB permissions

## Step 1: ALB Target Group (Instance Type)

```hcl
resource "aws_lb_target_group" "app" {
  name        = "${var.project_name}-app-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "instance"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
    protocol            = "HTTP"
    port                = "traffic-port"
  }

  # Stickiness for session affinity
  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400  # 1 day in seconds
    enabled         = true
  }

  # Deregistration delay - wait for in-flight requests to complete
  deregistration_delay = 30

  tags = {
    Name = "${var.project_name}-app-tg"
  }
}
```

## Step 2: IP-Type Target Group for ECS/Fargate

```hcl
resource "aws_lb_target_group" "fargate" {
  name        = "${var.project_name}-fargate-tg"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"  # Required for Fargate tasks

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200-299"
    protocol            = "HTTP"
  }

  # Use least outstanding requests for better distribution with variable response times
  load_balancing_algorithm_type = "least_outstanding_requests"

  tags = {
    Name = "${var.project_name}-fargate-tg"
  }
}
```

## Step 3: NLB Target Group with TCP Health Check

```hcl
resource "aws_lb_target_group" "nlb" {
  name        = "${var.project_name}-nlb-tg"
  port        = 443
  protocol    = "TLS"
  vpc_id      = var.vpc_id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 3
    unhealthy_threshold = 3
    interval            = 30
    protocol            = "HTTPS"
    path                = "/health"
    matcher             = "200"
    port                = "443"
  }

  # Preserve client IP address (NLB feature)
  preserve_client_ip = true

  tags = {
    Name = "${var.project_name}-nlb-tg"
  }
}
```

## Step 4: Lambda Target Group

```hcl
resource "aws_lb_target_group" "lambda" {
  name        = "${var.project_name}-lambda-tg"
  target_type = "lambda"

  # Health checks are managed through Lambda's own execution success
  health_check {
    enabled = false
  }

  tags = {
    Name = "${var.project_name}-lambda-tg"
  }
}

# Allow ALB to invoke Lambda

resource "aws_lambda_permission" "alb" {
  statement_id  = "AllowALBInvoke"
  action        = "lambda:InvokeFunction"
  function_name = var.lambda_function_name
  principal     = "elasticloadbalancing.amazonaws.com"
  source_arn    = aws_lb_target_group.lambda.arn
}

resource "aws_lb_target_group_attachment" "lambda" {
  target_group_arn = aws_lb_target_group.lambda.arn
  target_id        = var.lambda_function_arn
  depends_on       = [aws_lambda_permission.alb]
}
```

## Step 5: Register EC2 Instances as Targets

```hcl
resource "aws_lb_target_group_attachment" "instances" {
  for_each = toset(var.instance_ids)

  target_group_arn = aws_lb_target_group.app.arn
  target_id        = each.value
  port             = 8080
}
```

## Step 6: Deploy

```bash
tofu init
tofu plan
tofu apply

# Check target health
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>

# Describe target group attributes
aws elbv2 describe-target-group-attributes \
  --target-group-arn <target-group-arn>
```

## Conclusion

Set `deregistration_delay` to a value slightly longer than your application's longest request timeout to ensure in-flight requests complete before a target is removed during deployments. Use `target_type = "ip"` for containerized workloads (ECS, EKS) since containers don't have stable instance IDs. The `least_outstanding_requests` algorithm outperforms round-robin for workloads with variable response times, such as ML inference endpoints or database-heavy operations.
