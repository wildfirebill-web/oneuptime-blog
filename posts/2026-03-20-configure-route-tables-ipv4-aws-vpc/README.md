# How to Configure Route Tables for IPv4 Traffic in AWS VPC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, VPC, Route Tables, IPv4, Routing, Networking

Description: Create and configure AWS VPC route tables for IPv4 traffic, associate them with subnets, and add routes for Internet Gateways, NAT Gateways, VPC peering, and VPN connections.

## Introduction

VPC route tables are the routing control plane for your network. Every subnet must be associated with exactly one route table. Understanding how to create, populate, and associate route tables is fundamental to VPC networking.

## Listing Current Route Tables

```bash
VPC_ID=vpc-0abc123def456

# List all route tables in the VPC
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[].{ID:RouteTableId,Name:Tags[?Key==`Name`]|[0].Value}' \
  --output table
```

## Understanding the Main Route Table

Every VPC has a **main route table** (the default). Subnets not explicitly associated with a custom route table use the main table. It's best practice to leave the main table minimal (VPC-local only) and create explicit tables for each subnet tier.

## Creating a Route Table

```bash
# Create a custom route table
RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)
```

## Adding Routes

```bash
# Route to Internet Gateway (for public subnets)
aws ec2 create-route \
  --route-table-id $RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0abc123

# Route to NAT Gateway (for private subnets)
aws ec2 create-route \
  --route-table-id $RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-0abc123

# Route to VPC Peering connection
aws ec2 create-route \
  --route-table-id $RT_ID \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-0abc123

# Route to Virtual Private Gateway (VPN)
aws ec2 create-route \
  --route-table-id $RT_ID \
  --destination-cidr-block 192.168.0.0/24 \
  --gateway-id vgw-0abc123
```

## Associating a Subnet with a Route Table

```bash
# Associate a subnet with this route table
aws ec2 associate-route-table \
  --route-table-id $RT_ID \
  --subnet-id subnet-0priv1a

# Verify the association
aws ec2 describe-route-tables \
  --route-table-ids $RT_ID \
  --query 'RouteTables[0].Associations[].SubnetId' \
  --output text
```

## Replacing an Existing Route

```bash
# Replace the default route (when switching from IGW to NAT for an existing subnet)
aws ec2 replace-route \
  --route-table-id $RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-0abc123
```

## Deleting a Route

```bash
aws ec2 delete-route \
  --route-table-id $RT_ID \
  --destination-cidr-block 10.1.0.0/16
```

## Conclusion

Maintain separate route tables per tier (public, private per-AZ). Public tables point `0.0.0.0/0` to the IGW; private tables per AZ point to that AZ's NAT Gateway for resilience. Use VPC peering and VGW routes for hybrid connectivity. Associate subnets explicitly to avoid accidental default table usage.
