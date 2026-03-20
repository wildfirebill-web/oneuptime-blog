# How to Configure AWS VPC IPAM for Centralized IPv4 Address Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, VPC IPAM, IPv4, Cloud Networking, Address Management, Terraform

Description: Set up AWS VPC IP Address Manager (IPAM) to centrally plan, track, and allocate IPv4 CIDR ranges across multiple AWS accounts and regions.

AWS VPC IPAM is a managed service that allows you to plan, track, and monitor IPv4 addresses across your AWS organization. It enforces CIDR allocation governance and prevents overlapping VPC ranges.

## Setting Up AWS IPAM

### Via AWS Console

1. Navigate to **VPC → IPAM** in the AWS Console
2. Click **Create IPAM**
3. Set the operating regions
4. Note: IPAM requires AWS Organizations for multi-account use

### Via AWS CLI

```bash
# Create an IPAM with operating regions
aws ec2 create-ipam \
  --operating-regions RegionName=us-east-1 RegionName=us-west-2 RegionName=eu-west-1 \
  --description "Enterprise IPv4 IPAM"

# Note the IPAM ID from the response: ipam-xxxxxxxx
```

## Creating an IPAM Pool Hierarchy

```bash
# Create the top-level regional pool (covering your entire private IP space)
aws ec2 create-ipam-pool \
  --ipam-id ipam-xxxxxxxx \
  --address-family ipv4 \
  --allocation-min-netmask-length 16 \
  --allocation-max-netmask-length 28 \
  --auto-import \
  --locale us-east-1 \
  --description "US East - Top Level Pool"

# Note the pool ID: ipam-pool-xxxxxxxx
POOL_ID=ipam-pool-xxxxxxxx

# Add the parent CIDR to the pool
aws ec2 provision-ipam-pool-cidr \
  --ipam-pool-id $POOL_ID \
  --cidr 10.0.0.0/8
```

## Creating Sub-Pools for Different Environments

```bash
# Create a sub-pool for production VPCs
aws ec2 create-ipam-pool \
  --ipam-id ipam-xxxxxxxx \
  --source-ipam-pool-id $POOL_ID \
  --address-family ipv4 \
  --allocation-min-netmask-length 16 \
  --allocation-max-netmask-length 24 \
  --description "Production VPCs" \
  --locale us-east-1

PROD_POOL_ID=ipam-pool-yyyyyyyy

# Allocate a CIDR range to the production sub-pool
aws ec2 provision-ipam-pool-cidr \
  --ipam-pool-id $PROD_POOL_ID \
  --cidr 10.100.0.0/12
```

## Creating a VPC Using IPAM

```bash
# Create a VPC that automatically allocates from the IPAM pool
aws ec2 create-vpc \
  --ipv4-ipam-pool-id $PROD_POOL_ID \
  --ipv4-netmask-length 16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc-east}]'

# AWS assigns a non-overlapping /16 from 10.100.0.0/12
```

## Terraform Configuration

```hcl
# main.tf — AWS IPAM with Terraform

resource "aws_vpc_ipam" "main" {
  description = "Enterprise IPAM"
  operating_regions {
    region_name = "us-east-1"
  }
  operating_regions {
    region_name = "us-west-2"
  }
}

resource "aws_vpc_ipam_pool" "root" {
  address_family = "ipv4"
  ipam_scope_id  = aws_vpc_ipam.main.private_default_scope_id
  description    = "Root IPv4 Pool"
}

resource "aws_vpc_ipam_pool_cidr" "root" {
  ipam_pool_id = aws_vpc_ipam_pool.root.id
  cidr         = "10.0.0.0/8"
}

resource "aws_vpc_ipam_pool" "production" {
  address_family      = "ipv4"
  ipam_scope_id       = aws_vpc_ipam.main.private_default_scope_id
  source_ipam_pool_id = aws_vpc_ipam_pool.root.id
  description         = "Production VPCs"
  locale              = "us-east-1"
  allocation_min_netmask_length = 16
  allocation_max_netmask_length = 24
}

resource "aws_vpc" "main" {
  ipv4_ipam_pool_id   = aws_vpc_ipam_pool.production.id
  ipv4_netmask_length = 16
  tags = { Name = "production-vpc" }
}
```

## Viewing IPAM Utilization

```bash
# Check pool utilization
aws ec2 get-ipam-pool-cidrs \
  --ipam-pool-id $PROD_POOL_ID \
  --query 'IpamPoolCidrs[*].{CIDR:Cidr,State:State}'

# Get allocations within a pool
aws ec2 get-ipam-pool-allocations \
  --ipam-pool-id $PROD_POOL_ID \
  --output table
```

AWS VPC IPAM is essential for multi-account organizations where uncoordinated VPC CIDR selection leads to VPC peering conflicts and Transit Gateway routing failures.
