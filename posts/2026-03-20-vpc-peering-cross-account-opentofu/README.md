# How to Set Up VPC Peering Across AWS Accounts with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, VPC Peering, Cross-Account, Networking, Infrastructure as Code, IAM

Description: Learn how to configure VPC peering connections between VPCs in different AWS accounts using OpenTofu with multi-provider configuration and cross-account IAM roles.

## Introduction

Cross-account VPC peering allows private connectivity between VPCs owned by different AWS accounts without traffic traversing the public internet. This is common in multi-account AWS organizations where services in separate accounts need to communicate.

## Prerequisites

- OpenTofu v1.6+
- IAM permissions in both the requester and accepter AWS accounts
- Non-overlapping CIDR blocks in both VPCs

## Step 1: Configure Providers for Both Accounts

```hcl
# Requester account provider (default)
provider "aws" {
  region = "us-east-1"
}

# Accepter account provider using a cross-account role
provider "aws" {
  alias  = "accepter"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::${var.accepter_account_id}:role/TerraformPeeringRole"
  }
}
```

## Step 2: Look Up VPC Details from Each Account

```hcl
# Data source for requester VPC
data "aws_vpc" "requester" {
  id = var.requester_vpc_id
}

# Data source for accepter VPC from the other account
data "aws_vpc" "accepter" {
  provider = aws.accepter
  id       = var.accepter_vpc_id
}
```

## Step 3: Create the Peering Connection from the Requester

```hcl
# The requester account initiates the connection
resource "aws_vpc_peering_connection" "cross_account" {
  vpc_id        = var.requester_vpc_id
  peer_vpc_id   = var.accepter_vpc_id
  peer_owner_id = var.accepter_account_id
  peer_region   = var.accepter_region
  auto_accept   = false  # Must be false for cross-account

  tags = {
    Name        = "cross-account-peering"
    Requester   = data.aws_caller_identity.requester.account_id
    Accepter    = var.accepter_account_id
  }
}

data "aws_caller_identity" "requester" {}
```

## Step 4: Accept the Peering Request in the Accepter Account

```hcl
# The accepter account must explicitly accept the request
resource "aws_vpc_peering_connection_accepter" "cross_account" {
  provider                  = aws.accepter
  vpc_peering_connection_id = aws_vpc_peering_connection.cross_account.id
  auto_accept               = true

  tags = {
    Name = "cross-account-peering-accepter"
    Side = "Accepter"
  }
}
```

## Step 5: Enable DNS Resolution in Both VPCs

```hcl
# Enable DNS on the requester side
resource "aws_vpc_peering_connection_options" "requester" {
  vpc_peering_connection_id = aws_vpc_peering_connection.cross_account.id
  requester {
    allow_remote_vpc_dns_resolution = true
  }
  depends_on = [aws_vpc_peering_connection_accepter.cross_account]
}

# Enable DNS on the accepter side
resource "aws_vpc_peering_connection_options" "accepter" {
  provider                  = aws.accepter
  vpc_peering_connection_id = aws_vpc_peering_connection.cross_account.id
  accepter {
    allow_remote_vpc_dns_resolution = true
  }
  depends_on = [aws_vpc_peering_connection_accepter.cross_account]
}
```

## Step 6: Update Route Tables in Both Accounts

```hcl
# Route in requester VPC pointing to accepter VPC CIDR
resource "aws_route" "requester_to_accepter" {
  route_table_id            = var.requester_route_table_id
  destination_cidr_block    = data.aws_vpc.accepter.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.cross_account.id
}

# Route in accepter VPC pointing to requester VPC CIDR
resource "aws_route" "accepter_to_requester" {
  provider                  = aws.accepter
  route_table_id            = var.accepter_route_table_id
  destination_cidr_block    = data.aws_vpc.requester.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.cross_account.id
}
```

## Step 7: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

You have configured cross-account VPC peering using OpenTofu's multi-provider pattern. The accepter account role must allow `sts:AssumeRole` from the requester account for this to work. Remember to update security groups in both VPCs to allow traffic from the peer CIDR range, as security group rules are not automatically updated.
