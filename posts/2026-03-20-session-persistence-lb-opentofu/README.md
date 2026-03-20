# How to Configure Session Persistence with Load Balancers in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Load Balancer, Session Persistence, Sticky Sessions, OpenTofu, AWS, Azure, GCP

Description: Learn how to configure session persistence (sticky sessions) with load balancers using OpenTofu to ensure client requests are consistently routed to the same backend instance.

## Overview

Session persistence ensures that a client's requests are consistently routed to the same backend instance for the duration of a session. OpenTofu configures sticky sessions across AWS ALB, Azure Application Gateway, and GCP Load Balancer with appropriate cookie handling.

## Step 1: AWS ALB Sticky Sessions

```hcl
# main.tf - ALB with sticky sessions
resource "aws_lb_target_group" "sticky" {
  name     = "app-tg-sticky"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = module.vpc.vpc_id

  # Application-based stickiness (uses existing app cookie)
  stickiness {
    enabled         = true
    type            = "app_cookie"
    cookie_name     = "SESSIONID"  # Your application's session cookie
    cookie_duration = 86400        # 24 hours
  }

  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    matcher             = "200"
  }
}

# ALB-generated cookie stickiness (simpler - no app changes needed)
resource "aws_lb_target_group" "lb_cookie_sticky" {
  name     = "app-tg-lb-sticky"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = module.vpc.vpc_id

  stickiness {
    enabled  = true
    type     = "lb_cookie"   # ALB generates AWSALB cookie
    cookie_duration = 3600   # 1 hour
  }
}
```

## Step 2: Azure Application Gateway Sticky Sessions

```hcl
# Azure Application Gateway with cookie-based affinity
resource "azurerm_application_gateway" "sticky" {
  name                = "sticky-app-gateway"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  sku {
    name     = "Standard_v2"
    tier     = "Standard_v2"
    capacity = 2
  }

  backend_http_settings {
    name                  = "sticky-settings"
    port                  = 8080
    protocol              = "Http"
    cookie_based_affinity = "Enabled"   # Enable sticky sessions
    affinity_cookie_name  = "ApplicationGatewayAffinity"
    request_timeout       = 30

    probe_name = "app-probe"
  }

  # Custom affinity cookie name
  backend_http_settings {
    name                  = "custom-cookie-settings"
    port                  = 8080
    protocol              = "Http"
    cookie_based_affinity = "Enabled"
    affinity_cookie_name  = "AppSession"  # Custom cookie name
    request_timeout       = 60
  }
}
```

## Step 3: GCP Backend Service Session Affinity

```hcl
# GCP Load Balancer with session affinity
resource "google_compute_backend_service" "sticky" {
  name        = "sticky-backend"
  protocol    = "HTTP"
  timeout_sec = 30

  # Client IP-based affinity
  session_affinity = "CLIENT_IP"  # or GENERATED_COOKIE

  # Cookie-based affinity settings
  consistent_hash {
    http_cookie {
      name = "GCLB_SESSION"
      ttl {
        seconds = 3600
      }
    }

    minimum_ring_size = 1024  # For consistent hash load balancing
  }

  backend {
    group = google_compute_instance_group_manager.app.instance_group
  }

  health_checks = [google_compute_health_check.app.id]
}

# Header-based affinity (for gRPC or API services)
resource "google_compute_backend_service" "header_affinity" {
  name     = "header-affinity-backend"
  protocol = "HTTP"

  session_affinity = "HTTP_COOKIE"

  consistent_hash {
    http_header_name = "X-Session-Token"  # Route based on header
  }
}
```

## Step 4: ECS Service with Sticky Sessions

```hcl
# ECS service with ALB sticky sessions
resource "aws_ecs_service" "sticky_app" {
  name            = "sticky-app"
  cluster         = aws_ecs_cluster.app.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 6
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = module.vpc.private_subnets
    security_groups = [aws_security_group.app.id]
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.sticky.arn
    container_name   = "app"
    container_port   = 8080
  }

  # Deregistration delay must exceed session duration for graceful shutdown
  lifecycle {
    ignore_changes = [desired_count]
  }
}

# Longer deregistration delay when using sticky sessions
resource "aws_lb_target_group" "sticky_fargate" {
  name                 = "sticky-fargate-tg"
  port                 = 8080
  protocol             = "HTTP"
  vpc_id               = module.vpc.vpc_id
  target_type          = "ip"  # Required for Fargate

  deregistration_delay = 120  # Give in-flight sessions time to complete

  stickiness {
    enabled         = true
    type            = "lb_cookie"
    cookie_duration = 3600
  }
}
```

## Summary

Session persistence configured with OpenTofu routes clients to consistent backends using cookie-based affinity. Application-based stickiness (`app_cookie`) respects your existing session cookie for seamless integration, while load balancer-generated cookies (`lb_cookie`) require no application changes. The deregistration delay should exceed the session cookie duration to prevent active sessions from being dropped during deployments. For stateless applications, prefer distributed session storage (Redis, DynamoDB) over sticky sessions to avoid uneven load distribution when instances fail.
