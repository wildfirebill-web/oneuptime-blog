# How to Create an Application Load Balancer with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, ALB, Load Balancer, Infrastructure as Code, Networking

Description: Learn how to provision an AWS Application Load Balancer with listeners, target groups, and health checks using OpenTofu.

---

An AWS Application Load Balancer (ALB) distributes HTTP/HTTPS traffic to backend targets based on rules. OpenTofu lets you define the entire ALB configuration — listeners, target groups, and health checks — as version-controlled infrastructure.

---

## Create the Load Balancer

```hcl
resource "aws_lb" "main" {
  name               = "main-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = false

  tags = {
    Name        = "main-alb"
    Environment = "production"
  }
}
```

---

## Create a Target Group

```hcl
resource "aws_lb_target_group" "web" {
  name        = "web-tg"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"  # For ECS/Fargate; use "instance" for EC2

  health_check {
    enabled             = true
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }
}
```

---

## Create HTTP Listener with HTTPS Redirect

```hcl
resource "aws_lb_listener" "http" {
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

---

## Create HTTPS Listener

```hcl
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}
```

---

## Path-Based Routing Rule

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

---

## Output the ALB DNS Name

```hcl
output "alb_dns" {
  value = aws_lb.main.dns_name
}
```

---

## Summary

Create `aws_lb` for the ALB, `aws_lb_target_group` with health checks, and `aws_lb_listener` resources for HTTP (redirect to HTTPS) and HTTPS (forward to target group). Add `aws_lb_listener_rule` for path-based or host-based routing. Use `aws_acm_certificate` for TLS and output the `dns_name` for DNS CNAME configuration.
