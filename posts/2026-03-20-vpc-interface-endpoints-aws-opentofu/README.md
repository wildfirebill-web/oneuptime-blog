# How to Configure VPC Interface Endpoints for AWS Services with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, VPC, Interface Endpoints, PrivateLink, ECR, SSM, Secrets Manager, Infrastructure as Code

Description: Learn how to create AWS VPC Interface endpoints using OpenTofu for ECR, SSM, Secrets Manager, STS, and other AWS services to eliminate NAT gateway dependency for private workloads.

---

VPC Interface endpoints (powered by PrivateLink) provide private connectivity to AWS services via elastic network interfaces in your subnet. Unlike gateway endpoints, they cost ~$7.20/month per AZ but support many more services and enable private DNS resolution. They're essential for EKS clusters and private workloads that need to avoid NAT gateways.

## Interface Endpoint vs Gateway Endpoint

| Feature | Gateway Endpoint | Interface Endpoint |
|---------|-----------------|-------------------|
| Services | S3, DynamoDB only | 100+ AWS services |
| Cost | Free | ~$7.20/mo per AZ |
| DNS | Route table prefix list | Private DNS (standard hostname) |
| Security Group | Not applicable | Required |
| Availability | Route table based | ENI per subnet |

## Security Group for All Interface Endpoints

```hcl
# endpoints_sg.tf

resource "aws_security_group" "vpc_endpoints" {
  name        = "${var.prefix}-vpc-endpoints"
  description = "Allow HTTPS from VPC to interface endpoints"
  vpc_id      = var.vpc_id

  ingress {
    description = "HTTPS from VPC"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [data.aws_vpc.main.cidr_block]
  }

  # No egress needed - endpoints only receive inbound connections

  tags = {
    Name        = "${var.prefix}-vpc-endpoints"
    Environment = var.environment
    ManagedBy   = "opentofu"
  }
}
```

## ECR Endpoints (Required for EKS)

```hcl
# ecr_endpoints.tf
# EKS nodes in private subnets need these endpoints to pull container images

resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "${var.prefix}-ecr-api-endpoint"
  }
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "${var.prefix}-ecr-dkr-endpoint"
  }
}
```

## SSM Endpoints (Required for Session Manager)

```hcl
# ssm_endpoints.tf
# Required for AWS Systems Manager Session Manager (no bastion host needed)

locals {
  ssm_services = {
    ssm         = "com.amazonaws.${var.region}.ssm"
    ssmmessages = "com.amazonaws.${var.region}.ssmmessages"
    ec2messages = "com.amazonaws.${var.region}.ec2messages"
  }
}

resource "aws_vpc_endpoint" "ssm" {
  for_each = local.ssm_services

  vpc_id              = var.vpc_id
  service_name        = each.value
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "${var.prefix}-${each.key}-endpoint"
  }
}
```

## Secrets Manager and STS Endpoints

```hcl
# secrets_sts_endpoints.tf
resource "aws_vpc_endpoint" "secretsmanager" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "${var.prefix}-secretsmanager-endpoint"
  }
}

resource "aws_vpc_endpoint" "sts" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.sts"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "${var.prefix}-sts-endpoint"
  }
}
```

## All EKS-Required Endpoints in One Module

```hcl
# eks_endpoints.tf - complete set for fully private EKS clusters
locals {
  eks_interface_endpoints = {
    ec2              = "com.amazonaws.${var.region}.ec2"
    ecr_api          = "com.amazonaws.${var.region}.ecr.api"
    ecr_dkr          = "com.amazonaws.${var.region}.ecr.dkr"
    sts              = "com.amazonaws.${var.region}.sts"
    logs             = "com.amazonaws.${var.region}.logs"
    ssm              = "com.amazonaws.${var.region}.ssm"
    ssmmessages      = "com.amazonaws.${var.region}.ssmmessages"
    ec2messages      = "com.amazonaws.${var.region}.ec2messages"
    elasticloadbalancing = "com.amazonaws.${var.region}.elasticloadbalancing"
    autoscaling      = "com.amazonaws.${var.region}.autoscaling"
  }
}

resource "aws_vpc_endpoint" "eks_required" {
  for_each = local.eks_interface_endpoints

  vpc_id              = var.vpc_id
  service_name        = each.value
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name        = "${var.prefix}-${replace(each.key, "_", "-")}-endpoint"
    Environment = var.environment
    ManagedBy   = "opentofu"
  }
}
```

## Cost Optimization: Conditional Endpoints

```hcl
# Only create expensive interface endpoints when needed
variable "enable_interface_endpoints" {
  type = object({
    ssm            = bool
    secretsmanager = bool
    ecr            = bool
    sts            = bool
    logs           = bool
  })
  default = {
    ssm            = true
    secretsmanager = true
    ecr            = true
    sts            = true
    logs           = false  # $21.60/mo for 3 AZs - only enable if needed
  }
}
```

## Best Practices

- Create all three SSM endpoints (`ssm`, `ssmmessages`, `ec2messages`) together - all three are required for Session Manager to function; missing any one causes silent connection failures.
- Enable `private_dns_enabled = true` on all interface endpoints - this allows applications to use standard AWS service hostnames without configuration changes.
- Place interface endpoints in private subnets, not public subnets - endpoints are used by private resources, and placing them in public subnets doesn't add security or functionality.
- Calculate ROI before creating interface endpoints - each endpoint costs ~$7.20/month per AZ. At 3 AZs, that's $21.60/month per endpoint. Endpoints break even when they eliminate more than $21.60/month in NAT gateway data processing costs.
- For EKS, start with the S3 gateway endpoint (free) and ECR interface endpoints (required for image pulls) - then add others as needed rather than creating all endpoints upfront.
