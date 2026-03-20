# How to Configure AWS NLB with IPv6 Using Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, Terraform, IPv6, NLB, Load Balancer, Networking

Description: A guide to creating an AWS Network Load Balancer with IPv6 support using Terraform for TCP/UDP workloads.

The AWS Network Load Balancer (NLB) operates at Layer 4 and supports both IPv4 and IPv6 via its `dualstack` IP address type. NLBs are ideal for TCP/UDP workloads requiring ultra-low latency, static IPs, or TLS passthrough.

## Step 1: Create a VPC and Public Subnets with IPv6

```hcl
# vpc.tf - VPC with IPv6 CIDR

resource "aws_vpc" "main" {
  cidr_block                       = "10.0.0.0/16"
  assign_generated_ipv6_cidr_block = true
  tags = { Name = "main-vpc" }
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  ipv6_cidr_block   = cidrsubnet(aws_vpc.main.ipv6_cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  assign_ipv6_address_on_creation = true
  tags = { Name = "public-${count.index}" }
}
```

## Step 2: Create the Dual-Stack NLB

```hcl
# nlb.tf - Network Load Balancer with dualstack IPv6
resource "aws_lb" "nlb" {
  name               = "main-nlb"
  internal           = false
  load_balancer_type = "network"

  # Subnets with IPv6 CIDRs
  subnets = aws_subnet.public[*].id

  # Enable both IPv4 and IPv6
  ip_address_type = "dualstack"

  # NLBs do not use security groups - traffic filtering is at the subnet/NACL level
  enable_cross_zone_load_balancing = true

  tags = {
    Name = "main-nlb"
  }
}

output "nlb_dns_name" {
  value = aws_lb.nlb.dns_name
}
```

## Step 3: Create a TCP Target Group

```hcl
# target-group.tf - TCP target group for NLB
resource "aws_lb_target_group" "tcp" {
  name     = "tcp-tg"
  port     = 443
  protocol = "TCP"
  vpc_id   = aws_vpc.main.id

  # For client IP preservation with NLB, use IP target type
  target_type = "ip"

  health_check {
    protocol            = "TCP"
    port                = "traffic-port"
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 10
    interval            = 30
  }

  tags = {
    Name = "tcp-target-group"
  }
}
```

## Step 4: Create a TCP Listener

```hcl
# listener.tf - TCP listener on port 443
resource "aws_lb_listener" "tcp_443" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = 443
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tcp.arn
  }
}
```

## Step 5: Register EC2 Instance Targets

```hcl
# target-registration.tf - Register instances by IP address
resource "aws_lb_target_group_attachment" "app" {
  count            = length(aws_instance.app)
  target_group_arn = aws_lb_target_group.tcp.arn

  # Use the instance's private IPv4 (or IPv6 if target_type = "ip" and IPv6 is configured)
  target_id = aws_instance.app[count.index].private_ip
  port      = 443
}
```

## Step 6: Verify IPv6 Resolution

```bash
terraform apply

# Check NLB DNS resolves to both A and AAAA records
NLB_DNS=$(terraform output -raw nlb_dns_name)
dig A "$NLB_DNS"
dig AAAA "$NLB_DNS"

# Test TCP connectivity over IPv6
curl -6 --resolve "$NLB_DNS:443:[<AAAA-IP>]" https://$NLB_DNS/ -k -v
```

## IPv6 Client IP Preservation

NLBs preserve the client's source IP. For IPv6 clients:
- **Target type `instance`**: Source IP preserved at the instance
- **Target type `ip`**: Also preserved; ensure your application handles IPv6 client IPs

NLBs are the recommended choice for latency-sensitive TCP/UDP workloads that need IPv6 support, providing static Elastic IP assignment and full client IP preservation.
