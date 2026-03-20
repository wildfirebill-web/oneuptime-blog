# How to Configure IPv6 Subnets in AWS VPC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, IPv6, VPC, Subnets, Cloud Networking, Dual-Stack

Description: Create and configure IPv6-enabled subnets in AWS VPC, assign /64 CIDR blocks, and configure auto-assignment of IPv6 addresses to EC2 instances.

## Introduction

After enabling IPv6 on your VPC, you assign `/64` IPv6 CIDR blocks to individual subnets from the VPC's `/56` prefix. Each subnet gets a unique `/64` block. You can configure subnets to automatically assign IPv6 addresses to EC2 instances at launch, enabling dual-stack or IPv6-only operation.

## Assign IPv6 CIDR to Existing Subnet

```bash
# Get VPC IPv6 CIDR block

VPC_ID="vpc-12345678"

IPV6_CIDR=$(aws ec2 describe-vpcs \
    --vpc-ids "$VPC_ID" \
    --query "Vpcs[0].Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock" \
    --output text)

echo "VPC IPv6 CIDR: $IPV6_CIDR"
# Example: 2600:1f14:abc:de00::/56

# Get a subnet ID
SUBNET_ID="subnet-12345678"

# Assign IPv6 /64 (using subnet number 0x00 to 0xff)
# Replace the last two octets of /56 with the subnet number
IPV6_SUBNET="${IPV6_CIDR%00::/56}00::/64"
echo "Assigning IPv6 subnet: $IPV6_SUBNET"

aws ec2 associate-subnet-cidr-block \
    --subnet-id "$SUBNET_ID" \
    --ipv6-cidr-block "$IPV6_SUBNET"

# Enable auto-assignment of IPv6 addresses
aws ec2 modify-subnet-attribute \
    --subnet-id "$SUBNET_ID" \
    --assign-ipv6-address-on-creation
```

## Terraform Configuration for IPv6 Subnets

```hcl
# subnets.tf - IPv6-enabled subnets

locals {
  # Extract the /56 prefix to carve /64 subnets
  vpc_ipv6_cidr = aws_vpc.main.ipv6_cidr_block
}

resource "aws_subnet" "public_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  # Assign /64 from the VPC /56
  ipv6_cidr_block = cidrsubnet(aws_vpc.main.ipv6_cidr_block, 8, 1)

  # Auto-assign IPv6 to instances launched in this subnet
  assign_ipv6_address_on_creation = true

  # For public subnets, also map public IPv4
  map_public_ip_on_launch = true

  tags = { Name = "public-a-dual-stack" }
}

resource "aws_subnet" "public_b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"

  ipv6_cidr_block = cidrsubnet(aws_vpc.main.ipv6_cidr_block, 8, 2)
  assign_ipv6_address_on_creation = true
  map_public_ip_on_launch = true

  tags = { Name = "public-b-dual-stack" }
}

resource "aws_subnet" "private_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.10.0/24"
  availability_zone = "us-east-1a"

  # Private subnets also get IPv6 (all IPv6 in AWS is public)
  ipv6_cidr_block = cidrsubnet(aws_vpc.main.ipv6_cidr_block, 8, 10)
  assign_ipv6_address_on_creation = true

  tags = { Name = "private-a-dual-stack" }
}
```

## IPv6-Only Subnet Configuration

```hcl
# IPv6-only subnet (no IPv4 CIDR)
resource "aws_subnet" "ipv6_only" {
  vpc_id = aws_vpc.main.id

  # No IPv4 CIDR block
  ipv6_cidr_block = cidrsubnet(aws_vpc.main.ipv6_cidr_block, 8, 20)

  # Enable IPv6-only mode
  enable_resource_name_dns_aaaa_record_on_launch = true
  ipv6_native = true  # IPv6-only subnet

  availability_zone = "us-east-1a"

  tags = { Name = "ipv6-only-subnet" }
}
```

## Verify Subnet IPv6 Configuration

```bash
# List all subnets with IPv6 information
aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query "Subnets[*].{SubnetId:SubnetId, IPv4:CidrBlock, IPv6:Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock, AutoAssign:AssignIpv6AddressOnCreation, AZ:AvailabilityZone}" \
    --output table

# Check a specific subnet
aws ec2 describe-subnets \
    --subnet-ids "$SUBNET_ID" \
    --query "Subnets[0].{IPv6CIDR:Ipv6CidrBlockAssociationSet, AutoIPv6:AssignIpv6AddressOnCreation}"
```

## The cidrsubnet Function for /64 Carving

```bash
# AWS VPC gets a /56. To carve /64 subnets:
# /56 has room for 256 x /64 subnets (2^(64-56) = 256)

# Terraform cidrsubnet(ipv6_cidr, 8, n) works because:
# 64 - 56 = 8 additional bits needed
# n = subnet index (0-255)

# Example: VPC = 2600:1f14:abc:de00::/56
# Subnet 0: 2600:1f14:abc:de00::/64
# Subnet 1: 2600:1f14:abc:de01::/64
# Subnet 10: 2600:1f14:abc:de0a::/64
# Subnet 255: 2600:1f14:abc:deff::/64
```

## Conclusion

AWS IPv6 subnets use `/64` blocks carved from the VPC's `/56` block. Use `cidrsubnet(vpc.ipv6_cidr_block, 8, N)` in Terraform to generate each subnet's IPv6 CIDR. Enable `assign_ipv6_address_on_creation = true` to automatically assign IPv6 addresses to instances. Remember that all IPv6 addresses in AWS are globally routable - "private" vs "public" is controlled by routing and security groups, not by the IP address itself. Configure route tables to route `::/0` through an Internet Gateway for public subnets or an Egress-Only IGW for private subnets.
