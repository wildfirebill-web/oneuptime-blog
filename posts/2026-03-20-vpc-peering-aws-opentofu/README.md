# How to Set Up VPC Peering with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, VPC Peering, Networking, Infrastructure as Code

Description: Learn how to establish AWS VPC peering connections with OpenTofu to enable private network communication between VPCs in the same or different accounts.

## Introduction

VPC Peering creates a direct networking connection between two VPCs, allowing resources in each to communicate privately using their private IP addresses. OpenTofu automates the peering request, acceptance, and route table updates.

## Same-Account VPC Peering

```hcl
# Create the peering connection request
resource "aws_vpc_peering_connection" "app_to_shared" {
  vpc_id      = aws_vpc.app.id         # Requester VPC
  peer_vpc_id = aws_vpc.shared.id      # Accepter VPC
  auto_accept = true                   # Auto-accept when both VPCs are in same account

  tags = {
    Name = "app-to-shared-peering"
    Side = "requester"
  }
}

# Add route in app VPC to reach shared VPC CIDR via the peering connection
resource "aws_route" "app_to_shared" {
  count                     = length(aws_route_table.app_private)
  route_table_id            = aws_route_table.app_private[count.index].id
  destination_cidr_block    = aws_vpc.shared.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.app_to_shared.id
}

# Add route in shared VPC to reach app VPC CIDR
resource "aws_route" "shared_to_app" {
  count                     = length(aws_route_table.shared_private)
  route_table_id            = aws_route_table.shared_private[count.index].id
  destination_cidr_block    = aws_vpc.app.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.app_to_shared.id
}
```

## Cross-Account VPC Peering

```hcl
# In the requester account
provider "aws" {
  alias   = "requester"
  region  = "us-east-1"
  profile = "requester-account"
}

# In the accepter account
provider "aws" {
  alias   = "accepter"
  region  = "us-east-1"
  profile = "accepter-account"
}

resource "aws_vpc_peering_connection" "cross_account" {
  provider    = aws.requester
  vpc_id      = var.requester_vpc_id
  peer_vpc_id = var.accepter_vpc_id
  peer_owner_id = var.accepter_account_id
  auto_accept = false  # Cannot auto-accept cross-account

  tags = { Name = "cross-account-peering" }
}

# The accepter must explicitly accept the connection
resource "aws_vpc_peering_connection_accepter" "cross_account" {
  provider                  = aws.accepter
  vpc_peering_connection_id = aws_vpc_peering_connection.cross_account.id
  auto_accept               = true

  tags = { Name = "cross-account-peering-accepter" }
}
```

## DNS Resolution Across Peers

```hcl
# Enable DNS resolution in the peering connection
resource "aws_vpc_peering_connection_options" "requester" {
  vpc_peering_connection_id = aws_vpc_peering_connection.app_to_shared.id

  requester {
    allow_remote_vpc_dns_resolution = true
  }
}

resource "aws_vpc_peering_connection_options" "accepter" {
  vpc_peering_connection_id = aws_vpc_peering_connection.app_to_shared.id

  accepter {
    allow_remote_vpc_dns_resolution = true
  }
}
```

## Conclusion

VPC Peering is ideal for connecting a small number of VPCs (2–5). For hub-and-spoke architectures with many VPCs, consider AWS Transit Gateway instead—it scales better and avoids the complexity of full-mesh peering route tables.
