# How to Set Up AWS Direct Connect with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Direct Connect, Hybrid Cloud, Private Connectivity, Infrastructure as Code

Description: Learn how to configure AWS Direct Connect resources with OpenTofu, including virtual interfaces, BGP configuration, and Direct Connect Gateway for multi-VPC connectivity.

## Introduction

AWS Direct Connect provides dedicated private network connectivity between on-premises data centers and AWS, bypassing the public internet for more consistent network performance, lower latency, and higher bandwidth. OpenTofu manages the AWS side of the connection—VIFs, BGP configuration, and Direct Connect Gateways—while the physical circuit provisioning is handled by AWS and colocation providers.

## Prerequisites

- OpenTofu v1.6+
- An existing Direct Connect connection (physical circuit) from AWS or a partner
- AWS credentials with Direct Connect and VPC permissions

## Step 1: Create Direct Connect Gateway

```hcl
# DX Gateway enables connectivity to multiple VPCs across regions and accounts
resource "aws_dx_gateway" "main" {
  name            = "${var.project_name}-dx-gateway"
  amazon_side_asn = "64512"  # AWS-side BGP ASN (private range: 64512-65534)

  tags = {
    Name = "${var.project_name}-dx-gateway"
  }
}
```

## Step 2: Create Private Virtual Interface

```hcl
# Private VIF connects to a VPC via VGW or DX Gateway
resource "aws_dx_private_virtual_interface" "main" {
  connection_id    = var.dx_connection_id  # Physical DX connection ID
  name             = "${var.project_name}-private-vif"
  vlan             = 100                   # 802.1Q VLAN tag
  address_family   = "ipv4"
  bgp_asn          = 65001                 # Customer-side BGP ASN

  # BGP peering addresses
  amazon_address   = "169.254.100.1/30"   # AWS-side BGP peer IP
  customer_address = "169.254.100.2/30"   # On-premises BGP peer IP

  bgp_auth_key     = var.bgp_auth_key  # MD5 authentication key

  # Connect to DX Gateway (for multi-VPC access)
  dx_gateway_id    = aws_dx_gateway.main.id

  tags = {
    Name = "${var.project_name}-private-vif"
  }
}
```

## Step 3: Connect DX Gateway to VPCs

```hcl
# Attach DX Gateway to VGW in each VPC
resource "aws_dx_gateway_association" "vpc_1" {
  dx_gateway_id         = aws_dx_gateway.main.id
  associated_gateway_id = var.vgw_vpc_1_id  # Virtual Private Gateway of VPC 1

  allowed_prefixes = [
    "10.0.0.0/8",   # Allow on-premises routes to VPC 1
    "172.16.0.0/12"
  ]
}

resource "aws_dx_gateway_association" "vpc_2" {
  dx_gateway_id         = aws_dx_gateway.main.id
  associated_gateway_id = var.vgw_vpc_2_id

  allowed_prefixes = [
    "10.0.0.0/8",
    "172.16.0.0/12"
  ]
}
```

## Step 4: Create Transit Virtual Interface for Transit Gateway

```hcl
# Transit VIF connects to AWS Transit Gateway via DX Gateway
resource "aws_dx_transit_virtual_interface" "main" {
  connection_id  = var.dx_connection_id
  name           = "${var.project_name}-transit-vif"
  vlan           = 200
  address_family = "ipv4"
  bgp_asn        = 65001

  amazon_address   = "169.254.200.1/30"
  customer_address = "169.254.200.2/30"
  bgp_auth_key     = var.bgp_auth_key

  dx_gateway_id = aws_dx_gateway.main.id

  tags = {
    Name = "${var.project_name}-transit-vif"
  }
}

# Associate DX Gateway with Transit Gateway
resource "aws_dx_gateway_association" "tgw" {
  dx_gateway_id         = aws_dx_gateway.main.id
  associated_gateway_id = var.transit_gateway_id

  allowed_prefixes = [
    "10.0.0.0/8"
  ]
}
```

## Step 5: Create LAG for Redundancy

```hcl
# Link Aggregation Group combines multiple connections for redundancy and bandwidth
resource "aws_dx_lag" "main" {
  name                  = "${var.project_name}-lag"
  connections_bandwidth = "10Gbps"
  location              = var.dx_location  # Direct Connect location code
  force_destroy         = false

  tags = {
    Name = "${var.project_name}-lag"
  }
}
```

## Step 6: Deploy

```bash
tofu init
tofu plan
tofu apply

# Check virtual interface state
aws directconnect describe-virtual-interfaces \
  --virtual-interface-id <vif-id> \
  --query 'virtualInterfaces[0].{State: virtualInterfaceState, BGP: bgpPeers[0].bgpStatus}'
```

## Conclusion

Direct Connect physical provisioning (ordering circuits) is done outside Terraform through AWS or partner portals; OpenTofu manages the logical configuration on top. For production Direct Connect setups, always deploy two VIFs on separate physical connections in separate colocation facilities for redundancy. Use a Direct Connect Gateway with a Transit Gateway attachment for the most scalable architecture—it allows a single DX connection to reach all VPCs across regions.
