# How to Configure AWS Client VPN IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, Client VPN, IPv6, OpenVPN, Remote Access, Dual-Stack, VPN

Description: Configure AWS Client VPN to assign IPv6 addresses to VPN clients and route IPv6 traffic over the VPN tunnel.

## Introduction

AWS Client VPN IPv6 enables private IPv6 connectivity between cloud resources and on-premises or inter-VPC networks. Proper configuration requires setting up dual-stack support, IPv6 BGP sessions, and route advertisement.

## Prerequisites

- VPC/VNet with dual-stack (IPv4 + IPv6) subnets
- An existing AWS account with appropriate IAM permissions
- IPv6 address space allocated for the connection

## Step 1: Verify IPv6 Prerequisites

```bash
# Check VPC has IPv6 CIDR

aws ec2 describe-vpcs --query 'Vpcs[].{VpcId:VpcId, IPv6CIDRs:Ipv6CidrBlockAssociationSet}'
```

## Step 2: Enable IPv6 on the Service

```bash
# Enable IPv6 for the relevant AWS service
# (Command varies by specific service)
aws ec2 describe-vpc-attribute     --vpc-id vpc-0123456789abcdef0     --attribute enableDnsSupport

# Associate IPv6 CIDR block
aws ec2 associate-vpc-cidr-block     --vpc-id vpc-0123456789abcdef0     --amazon-provided-ipv6-cidr-block
```

## Step 3: Configure IPv6 BGP

```bash
# Configure BGP for IPv6 on Direct Connect or VPN
# For Direct Connect Virtual Interface:
aws directconnect create-private-virtual-interface     --connection-id dxcon-XXXXX     --new-private-virtual-interface         virtualInterfaceName=ipv6-vif,        vlan=100,        asn=65000,        amazonAddress=,        customerAddress=,        addressFamily=ipv6,        virtualGatewayId=vgw-XXXXX
```

## Step 4: Add IPv6 Routes

```bash
# Add IPv6 route to VPC route table
aws ec2 create-route \
    --route-table-id rtb-0123456789abcdef0 \
    --destination-ipv6-cidr-block '::/0' \
    --gateway-id igw-XXXXX
```

## Step 5: Test IPv6 Connectivity

```bash
# Test from cloud instance
ping6 -c 3 <on-premises-ipv6-address>

# Verify route is learned
aws ec2 describe-route-tables --route-table-ids rtb-XXX --query 'RouteTables[].Routes[?DestinationIpv6CidrBlock]'
```

## Step 6: Terraform Example

```hcl
# Terraform for AWS Client VPN IPv6
resource "aws_vpn_connection" "ipv6_vpn" {
  vpn_gateway_id      = aws_vpn_gateway.main.id
  customer_gateway_id = aws_customer_gateway.onprem.id
  type                = "ipsec.1"

  # Enable IPv6
  local_ipv6_network_cidr  = "::/0"
  remote_ipv6_network_cidr = "::/0"
  tunnel_inside_ip_version = "ipv6"
}
```

## Conclusion

AWS Client VPN IPv6 requires enabling dual-stack at the subnet level, configuring IPv6 BGP sessions, and adding IPv6 routes in the relevant route tables. Test connectivity end-to-end after configuration. Use Terraform for declarative, repeatable deployments. Monitor IPv6 BGP session state and route advertisement with OneUptime's network health checks.
