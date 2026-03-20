# How to Configure Cross-Region VPC Peering for IPv4 in AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, VPC Peering, Cross-Region, IPv4, Networking, OpenTofu

Description: Learn how to create cross-region VPC peering connections in AWS to enable private IPv4 communication between VPCs in different regions.

---

Cross-region VPC peering connects VPCs in different AWS regions over Amazon's private network, enabling private IPv4 communication without internet transit. OpenTofu manages the peering connection, acceptance, and route configuration.

---

## Create the Peering Connection (Requester)

```hcl
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

resource "aws_vpc_peering_connection" "cross_region" {
  provider    = aws.us_east
  vpc_id      = aws_vpc.us_east.id
  peer_vpc_id = aws_vpc.eu_west.id
  peer_region = "eu-west-1"
  auto_accept = false

  tags = {
    Name = "us-east-to-eu-west-peering"
    Side = "requester"
  }
}
```

---

## Accept the Peering Connection (Accepter)

```hcl
resource "aws_vpc_peering_connection_accepter" "cross_region" {
  provider                  = aws.eu_west
  vpc_peering_connection_id = aws_vpc_peering_connection.cross_region.id
  auto_accept               = true

  tags = {
    Name = "us-east-to-eu-west-peering"
    Side = "accepter"
  }
}
```

---

## Add Routes for the Peered VPC (Both Sides)

```hcl
# Route in US East pointing to EU West

resource "aws_route" "us_to_eu" {
  provider                  = aws.us_east
  route_table_id            = aws_route_table.us_east_private.id
  destination_cidr_block    = aws_vpc.eu_west.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.cross_region.id
}

# Route in EU West pointing to US East
resource "aws_route" "eu_to_us" {
  provider                  = aws.eu_west
  route_table_id            = aws_route_table.eu_west_private.id
  destination_cidr_block    = aws_vpc.us_east.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.cross_region.id
}
```

---

## Update Security Groups

```hcl
# Allow traffic from the peer VPC CIDR
resource "aws_security_group_rule" "allow_peer" {
  provider          = aws.us_east
  type              = "ingress"
  from_port         = 0
  to_port           = 65535
  protocol          = "tcp"
  cidr_blocks       = [aws_vpc.eu_west.cidr_block]
  security_group_id = aws_security_group.app_us_east.id
}
```

---

## Verify Peering

```bash
# Check peering connection status
aws ec2 describe-vpc-peering-connections   --region us-east-1   --query 'VpcPeeringConnections[*].{Status:Status.Code,Id:VpcPeeringConnectionId}'

# Test from an instance in US East
ping <eu-west-private-ip>
```

---

## Summary

Create the peering connection in the requester region with `aws_vpc_peering_connection`, accept it in the peer region with `aws_vpc_peering_connection_accepter`, and add routes in both VPCs' route tables. Update security groups to allow traffic from the peer VPC CIDR. VPC CIDRs must not overlap for peering to work.
