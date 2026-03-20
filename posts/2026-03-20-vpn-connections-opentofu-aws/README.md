# How to Configure VPN Connections on AWS with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, VPN, Site-to-Site VPN, Networking, Infrastructure as Code

Description: Learn how to provision AWS Site-to-Site VPN connections using OpenTofu, including Virtual Private Gateways, Customer Gateways, and static route configuration.

## Introduction

AWS Site-to-Site VPN provides encrypted connectivity between your AWS VPC and on-premises networks. Managing VPN configuration with OpenTofu ensures it is version-controlled and reproducible across environments.

## Provider Configuration

```hcl
provider "aws" {
  region = var.aws_region
}
```

## Customer Gateway

The Customer Gateway represents your on-premises VPN device:

```hcl
resource "aws_customer_gateway" "on_prem" {
  bgp_asn    = 65000
  ip_address = var.on_prem_public_ip
  type       = "ipsec.1"

  tags = {
    Name        = "on-prem-gateway"
    Environment = var.environment
  }
}
```

## Virtual Private Gateway

```hcl
resource "aws_vpn_gateway" "main" {
  vpc_id          = aws_vpc.main.id
  amazon_side_asn = 64512

  tags = {
    Name = "main-vpn-gateway"
  }
}
```

## VPN Connection

```hcl
resource "aws_vpn_connection" "main" {
  customer_gateway_id = aws_customer_gateway.on_prem.id
  vpn_gateway_id      = aws_vpn_gateway.main.id
  type                = "ipsec.1"
  static_routes_only  = true

  tags = {
    Name        = "main-vpn-connection"
    Environment = var.environment
  }
}
```

## Static Route

```hcl
resource "aws_vpn_connection_route" "on_prem" {
  destination_cidr_block = var.on_prem_cidr
  vpn_connection_id      = aws_vpn_connection.main.id
}
```

## Route Propagation

Enable route propagation to automatically add VPN routes to route tables:

```hcl
resource "aws_vpn_gateway_route_propagation" "main" {
  route_table_id = aws_route_table.private.id
  vpn_gateway_id = aws_vpn_gateway.main.id
}
```

## Outputs for VPN Configuration

```hcl
output "vpn_tunnel1_address" {
  value = aws_vpn_connection.main.tunnel1_address
}

output "vpn_tunnel2_address" {
  value = aws_vpn_connection.main.tunnel2_address
}

output "vpn_tunnel1_psk" {
  value     = aws_vpn_connection.main.tunnel1_preshared_key
  sensitive = true
}
```

## Deploying

```bash
tofu init
tofu plan
tofu apply
```

After deployment, use the tunnel addresses and PSKs from the outputs to configure your on-premises VPN device.

## Verifying Tunnel Status

```bash
aws ec2 describe-vpn-connections \
  --vpn-connection-ids <vpn-id> \
  --query 'VpnConnections[0].VgwTelemetry[*].{Status:Status,IP:OutsideIpAddress}'
```

## Conclusion

OpenTofu simplifies AWS VPN connection management by codifying the Customer Gateway, Virtual Private Gateway, and connection resources. The outputs provide the tunnel information needed to configure the on-premises side, completing the VPN setup as part of a single `tofu apply`.
