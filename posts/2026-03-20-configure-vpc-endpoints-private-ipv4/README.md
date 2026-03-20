# How to Configure VPC Endpoints for Private IPv4 Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, VPC Endpoints, PrivateLink, IPv4, Networking, OpenTofu

Description: Learn how to create Gateway and Interface VPC Endpoints to access AWS services like S3 and SSM from private subnets without internet access.

---

VPC Endpoints let resources in private subnets communicate with AWS services without routing traffic through the internet. Gateway endpoints (S3, DynamoDB) are free; Interface endpoints use PrivateLink and have per-hour charges.

---

## Gateway Endpoint for S3

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = aws_route_table.private[*].id

  tags = {
    Name = "s3-vpc-endpoint"
  }
}
```

Gateway endpoints add a route entry to your route tables automatically.

---

## Interface Endpoint for SSM

```hcl
resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.us-east-1.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "ssm-vpc-endpoint"
  }
}

resource "aws_vpc_endpoint" "ssmmessages" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.us-east-1.ssmmessages"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ec2messages" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.us-east-1.ec2messages"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}
```

---

## Security Group for Interface Endpoints

```hcl
resource "aws_security_group" "vpc_endpoints" {
  name   = "vpc-endpoints-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }
}
```

---

## Verify SSM Connectivity

```bash
# From a private EC2 instance (no public IP, no NAT)

aws ssm start-session --target i-0abc1234567890def
```

---

## Summary

Create Gateway endpoints for S3 and DynamoDB by specifying route table IDs - they're free and add automatic routes. Create Interface endpoints for other services (SSM, ECR, Secrets Manager) with subnet IDs, a security group allowing port 443 from the VPC CIDR, and `private_dns_enabled = true`. This lets private instances access AWS services without internet access or NAT Gateway costs.
