# How to Configure AWS NLB with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, IPv6, NLB, Network Load Balancer, Dualstack, Load Balancing, TCP

Description: Configure an AWS Network Load Balancer in dualstack mode for IPv6 TCP/UDP load balancing, including target group configuration and static IP assignments.

## Introduction

AWS Network Load Balancers support IPv6 through the same "dualstack" mechanism as ALBs, but with different capabilities: NLBs operate at Layer 4 (TCP/UDP/TLS), preserve client source IP addresses, and support static Elastic IP assignments per availability zone. NLB dualstack enables IPv6 clients to connect to TCP/UDP services that backend targets may serve over IPv4.

## Create Dualstack NLB

```bash
# Create NLB with dualstack mode
aws elbv2 create-load-balancer \
    --name my-dualstack-nlb \
    --type network \
    --scheme internet-facing \
    --ip-address-type dualstack \
    --subnets subnet-pub-a subnet-pub-b

# Or convert existing NLB to dualstack
NLB_ARN="arn:aws:elasticloadbalancing:us-east-1:123:loadbalancer/net/my-nlb/abc"
aws elbv2 set-ip-address-type \
    --load-balancer-arn "$NLB_ARN" \
    --ip-address-type dualstack

# Get the NLB DNS name (includes both A and AAAA records)
aws elbv2 describe-load-balancers \
    --load-balancer-arns "$NLB_ARN" \
    --query "LoadBalancers[0].{DNSName:DNSName, IpType:IpAddressType}"
```

## Terraform NLB with IPv6

```hcl
# nlb_ipv6.tf

resource "aws_lb" "nlb" {
  name               = "main-nlb"
  internal           = false
  load_balancer_type = "network"
  ip_address_type    = "dualstack"

  # NLBs in specific subnets
  subnet_mapping {
    subnet_id = aws_subnet.public_a.id
  }
  subnet_mapping {
    subnet_id = aws_subnet.public_b.id
  }

  # NLBs don't use security groups (unlike ALBs)
  # Access control is via security groups on target instances

  enable_deletion_protection = false

  tags = { Name = "main-nlb" }
}

# TCP target group
resource "aws_lb_target_group" "tcp_443" {
  name        = "tcp-targets-443"
  port        = 443
  protocol    = "TCP"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id

  # Client IP preservation (NLB feature)
  preserve_client_ip = "true"

  health_check {
    protocol            = "TCP"
    healthy_threshold   = 3
    unhealthy_threshold = 3
    interval            = 10
  }
}

# TLS listener (NLB handles TLS termination)
resource "aws_lb_listener" "tls" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = "443"
  protocol          = "TLS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tcp_443.arn
  }
}

# TCP listener (no TLS termination)
resource "aws_lb_listener" "tcp_80" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = "80"
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tcp_80.arn
  }
}
```

## NLB Source IP Preservation with IPv6

```bash
# NLBs preserve client source IPs by default
# Backend instances see the actual client IPv6 address

# On backend EC2 instances, check access logs:
# - Access log shows the real client IPv6 address
# - Security group on backend must allow IPv6 from the client

# Security group for NLB targets (must allow IPv6 client addresses)
resource "aws_security_group" "nlb_target" {
  vpc_id = aws_vpc.main.id

  # Allow from anywhere (NLB preserves source IP)
  ingress {
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
    description      = "NLB passes real client IPs"
  }
}
```

## UDP Load Balancing with IPv6 (NLB Only)

```hcl
# NLB supports UDP — ALB does not
resource "aws_lb_listener" "udp_dns" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = "53"
  protocol          = "UDP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.dns.arn
  }
}

resource "aws_lb_target_group" "dns" {
  name        = "dns-targets"
  port        = 53
  protocol    = "UDP"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id

  health_check {
    protocol = "TCP"
    port     = 53
  }
}
```

## Conclusion

AWS NLBs with `ip_address_type = "dualstack"` accept IPv6 client connections at Layer 4, enabling TCP/UDP services to serve IPv6 clients. Unlike ALBs, NLBs preserve client source IPs — backend instances see the actual client IPv6 address, so security groups on targets must allow IPv6 from `::/0` when using NLB. NLBs don't use security groups themselves. The NLB DNS name includes both A and AAAA records when dualstack is enabled, allowing Happy Eyeballs clients to prefer IPv6 connections.
