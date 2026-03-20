# How to Configure an Internet Gateway for IPv4 Traffic in AWS VPC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, VPC, Internet Gateway, IPv4, Networking, Cloud

Description: Create and attach an AWS Internet Gateway to a VPC, configure route tables to direct IPv4 traffic through the IGW, and verify that instances in public subnets can reach the internet.

## Introduction

An Internet Gateway (IGW) enables bidirectional IPv4 connectivity between a VPC and the internet. Instances in subnets routed through an IGW with a public IP can send and receive internet traffic. Without an IGW, VPC instances are completely isolated from the internet.

## Creating and Attaching an Internet Gateway

```bash
VPC_ID=vpc-0abc123def456

# Create the internet gateway

IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=prod-igw}]' \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

echo "IGW created: $IGW_ID"

# Attach to the VPC (one IGW per VPC)
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID
```

## Creating a Public Route Table

A subnet becomes public only when its route table points `0.0.0.0/0` to the IGW:

```bash
# Create a new route table for public subnets
PUB_RT=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

# Add the default route pointing to the IGW
aws ec2 create-route \
  --route-table-id $PUB_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID
```

## Associating Public Subnets with the Route Table

```bash
# Associate each public subnet with the public route table
for SUBNET_ID in subnet-0pub1a subnet-0pub1b; do
    aws ec2 associate-route-table \
      --route-table-id $PUB_RT \
      --subnet-id $SUBNET_ID
done
```

## Verifying the Route Table

```bash
# Confirm the route to IGW is present
aws ec2 describe-route-tables \
  --route-table-ids $PUB_RT \
  --query 'RouteTables[0].Routes' \
  --output table
```

Expected output:

```text
+------------------+-------------------+-------+
| DestinationCidr  | GatewayId         | State |
+------------------+-------------------+-------+
| 10.0.0.0/16      | local             | active|
| 0.0.0.0/0        | igw-0abc123def    | active|
+------------------+-------------------+-------+
```

## Testing Internet Connectivity from an EC2 Instance

After launching an instance in the public subnet with a public IP:

```bash
# SSH into the instance and test connectivity
ping -c 3 8.8.8.8
curl -s https://checkip.amazonaws.com
```

## Detaching and Deleting an IGW

```bash
# Detach before deleting
aws ec2 detach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID

# Delete the IGW
aws ec2 delete-internet-gateway \
  --internet-gateway-id $IGW_ID
```

## Conclusion

An IGW is the only path for direct internet connectivity in a VPC. Create it, attach it, then add `0.0.0.0/0 → IGW` to the route table for public subnets. Private subnets use NAT Gateways instead, which are placed in public subnets and provide one-way outbound internet access.
