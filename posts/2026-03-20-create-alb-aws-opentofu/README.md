# How to Create an Application Load Balancer with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, ALB, Load Balancer, Infrastructure as Code

Description: Learn how to create an AWS Application Load Balancer with OpenTofu, including HTTPS listeners, target groups, health checks, and redirect rules.

## Introduction

An Application Load Balancer (ALB) distributes HTTP/HTTPS traffic across multiple targets. OpenTofu manages the ALB, its listeners, target groups, and routing rules as code-making it easy to configure HTTPS with ACM certificates, path-based routing, and automatic HTTP-to-HTTPS redirects.

## Security Group for ALB

```hcl
resource "aws_security_group" "alb" {
  name        = "${var.name}-alb-sg"
  description = "Security group for ALB"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## Application Load Balancer

```hcl
resource "aws_lb" "main" {
  name               = "${var.name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnet_ids

  enable_deletion_protection = var.environment == "prod" ? true : false

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = var.name
    enabled = true
  }

  tags = {
    Name        = "${var.name}-alb"
    Environment = var.environment
  }
}
```

## Target Group

```hcl
resource "aws_lb_target_group" "app" {
  name        = "${var.name}-tg"
  port        = var.container_port
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"  # Use "ip" for ECS Fargate, "instance" for EC2

  health_check {
    enabled             = true
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    matcher             = "200-299"
  }

  deregistration_delay = 30

  tags = { Name = "${var.name}-tg" }
}
```

## HTTPS Listener (Port 443)

```hcl
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.acm_certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

## HTTP to HTTPS Redirect

```hcl
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"

    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

## Path-Based Routing Rules

```hcl
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}
```

## Outputs

```hcl
output "alb_arn"      { value = aws_lb.main.arn }
output "alb_dns_name" { value = aws_lb.main.dns_name }
output "alb_zone_id"  { value = aws_lb.main.zone_id }
output "target_group_arn" { value = aws_lb_target_group.app.arn }
```

## Conclusion

An ALB with HTTPS termination, HTTP redirect, and path-based routing is the standard front-door for AWS web applications. Enable access logging from the start for security auditing, and use `TLS 1.3` security policies to enforce modern encryption standards.
