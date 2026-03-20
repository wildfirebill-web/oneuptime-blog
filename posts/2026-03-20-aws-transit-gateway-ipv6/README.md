# How to Configure AWS Transit Gateway for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, Transit Gateway, IPv6, VPC, Cloud Networking, Dual-Stack, BGP

Description: Configure AWS Transit Gateway to route IPv6 traffic between dual-stack VPCs, on-premises networks, and the internet using dual-stack attachments and route tables.

## Introduction

AWS Transit Gateway (TGW) acts as a regional network hub connecting multiple VPCs and on-premises networks. It supports IPv6 through dual-stack attachments and route tables. Each VPC attachment can carry both IPv4 and IPv6 traffic.

## Step 1: Create a Transit Gateway with IPv6 Support

```bash
# Create TGW

TGW_ID=$(aws ec2 create-transit-gateway \
    --description "Dual-stack TGW" \
    --options \
        DefaultRouteTableAssociation=enable,\
        DefaultRouteTablePropagation=enable,\
        MulticastSupport=disable,\
        TransitGatewayAutoAcceptSharedAttachments=disable \
    --query 'TransitGateway.TransitGatewayId' \
    --output text)

echo "TGW ID: $TGW_ID"

# Wait for available state
aws ec2 wait transit-gateway-available --transit-gateway-ids $TGW_ID
```

## Step 2: Attach a Dual-Stack VPC

```bash
# Get VPC and subnet IDs
VPC_ID="vpc-0123456789abcdef0"
SUBNET_IDS="subnet-0123456789abcdef0,subnet-fedcba9876543210f"

# Create dual-stack TGW attachment
TGW_ATTACH_ID=$(aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id $TGW_ID \
    --vpc-id $VPC_ID \
    --subnet-ids $SUBNET_IDS \
    --options \
        DnsSupport=enable,\
        Ipv6Support=enable,\
        ApplianceModeSupport=disable \
    --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' \
    --output text)

echo "Attachment ID: $TGW_ATTACH_ID"
```

## Step 3: Configure IPv6 Route Tables

```bash
# Get default TGW route table
TGW_RT_ID=$(aws ec2 describe-transit-gateway-route-tables \
    --filters "Name=transit-gateway-id,Values=$TGW_ID" \
              "Name=state,Values=available" \
    --query 'TransitGatewayRouteTables[0].TransitGatewayRouteTableId' \
    --output text)

# Add static IPv6 default route
aws ec2 create-transit-gateway-route \
    --destination-cidr-block "::/0" \
    --transit-gateway-route-table-id $TGW_RT_ID \
    --transit-gateway-attachment-id $INTERNET_ATTACH_ID

# List IPv6 routes in TGW route table
aws ec2 search-transit-gateway-routes \
    --transit-gateway-route-table-id $TGW_RT_ID \
    --filters "Name=type,Values=static,propagated"
```

## Step 4: VPC Route Tables for IPv6

```bash
# Update VPC route table to route IPv6 traffic through TGW
VPC_RT_ID="rtb-0123456789abcdef0"

# Route all IPv6 to TGW (for VPC-to-VPC routing)
aws ec2 create-route \
    --route-table-id $VPC_RT_ID \
    --destination-ipv6-cidr-block "::/0" \
    --transit-gateway-id $TGW_ID

# Or route specific IPv6 prefix to TGW
aws ec2 create-route \
    --route-table-id $VPC_RT_ID \
    --destination-ipv6-cidr-block "2001:db8:peer-vpc::/48" \
    --transit-gateway-id $TGW_ID
```

## Step 5: Terraform Configuration

```hcl
# main.tf

resource "aws_ec2_transit_gateway" "main" {
  description = "Dual-stack TGW"
  
  default_route_table_association = "enable"
  default_route_table_propagation = "enable"

  tags = {
    Name = "main-tgw"
  }
}

resource "aws_ec2_transit_gateway_vpc_attachment" "vpc_a" {
  subnet_ids         = var.vpc_a_subnets
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = var.vpc_a_id

  ipv6_support = "enable"  # Enable IPv6 on attachment

  tags = {
    Name = "vpc-a-attachment"
  }
}

resource "aws_ec2_transit_gateway_route" "ipv6_default" {
  destination_cidr_block         = "::/0"
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.internet.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway.main.association_default_route_table_id
}
```

## Step 6: Verify IPv6 Connectivity

```bash
# Test IPv6 routing through TGW between VPCs
# From an EC2 instance in VPC A
ping6 -c 3 2001:db8:vpc-b::instance

# Check TGW route propagation
aws ec2 get-transit-gateway-route-table-propagations \
    --transit-gateway-route-table-id $TGW_RT_ID

# View IPv6 routes in TGW route table
aws ec2 search-transit-gateway-routes \
    --transit-gateway-route-table-id $TGW_RT_ID \
    --filters "Name=route-search.subnet-of-match,Values=::/0"
```

## Conclusion

AWS Transit Gateway enables IPv6 routing between VPCs by enabling `Ipv6Support=enable` on each VPC attachment and adding IPv6 routes (`::/0` or specific prefixes) to TGW and VPC route tables. Terraform's `ipv6_support = "enable"` makes this declarative. Monitor TGW attachment health and IPv6 route propagation with OneUptime's network connectivity checks between VPCs.
