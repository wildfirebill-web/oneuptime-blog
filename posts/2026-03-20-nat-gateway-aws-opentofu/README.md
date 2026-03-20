# How to Create a NAT Gateway with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, NAT Gateway, Networking, Infrastructure as Code

Description: Learn how to create AWS NAT Gateways with OpenTofu to give private subnet instances outbound internet access without exposing them to inbound connections.

## Introduction

A NAT Gateway allows instances in private subnets to initiate outbound connections to the internet while preventing inbound connections from the internet. For high availability, deploy one NAT Gateway per availability zone.

## Elastic IPs for NAT Gateways

```hcl
# One EIP per AZ for high-availability NAT Gateways
resource "aws_eip" "nat" {
  count  = var.az_count
  domain = "vpc"

  tags = {
    Name = "${var.name}-nat-eip-${count.index + 1}"
  }
}
```

## NAT Gateways

```hcl
resource "aws_nat_gateway" "main" {
  count         = var.az_count
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id  # NAT GW lives in PUBLIC subnet

  tags = {
    Name = "${var.name}-nat-${count.index + 1}"
  }

  depends_on = [aws_internet_gateway.main]
}
```

## Private Route Tables with NAT Gateway Routes

```hcl
resource "aws_route_table" "private" {
  count  = var.az_count
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.name}-private-rt-${count.index + 1}"
  }
}

resource "aws_route" "private_nat" {
  count                  = var.az_count
  route_table_id         = aws_route_table.private[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  # Route outbound traffic through the NAT Gateway in the same AZ
  nat_gateway_id         = aws_nat_gateway.main[count.index].id
}

resource "aws_route_table_association" "private" {
  count          = var.az_count
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

## Single NAT Gateway for Cost Savings (Dev/Test)

```hcl
# For non-production environments, use a single NAT gateway to reduce costs
resource "aws_nat_gateway" "single" {
  count         = var.environment == "prod" ? 0 : 1
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id

  tags = { Name = "${var.name}-nat-single" }
}

resource "aws_route" "private_single_nat" {
  count                  = var.environment != "prod" ? length(aws_subnet.private) : 0
  route_table_id         = aws_route_table.private[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.single[0].id
}
```

## Cost Comparison

| Configuration | Monthly Cost (est.) | Availability |
|---|---|---|
| 1 NAT Gateway | ~$32/month + data | Single AZ — not HA |
| 3 NAT Gateways (one per AZ) | ~$96/month + data | Fully HA |
| NAT Instance (t3.nano) | ~$3/month | Manually managed |

## Outputs

```hcl
output "nat_gateway_ids"  { value = aws_nat_gateway.main[*].id }
output "nat_public_ips"   { value = aws_eip.nat[*].public_ip }
```

## Conclusion

For production, deploy one NAT Gateway per AZ to ensure private instances remain reachable even if an AZ fails. For development and staging, a single NAT Gateway significantly reduces cost. The EIP addresses are stable—whitelist them in external systems that need to accept traffic from your instances.
