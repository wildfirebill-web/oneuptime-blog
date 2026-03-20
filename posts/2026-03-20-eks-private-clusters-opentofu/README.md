# How to Configure EKS Private Clusters with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, EKS, Private Cluster, VPC Endpoints, Security, Infrastructure as Code, Kubernetes

Description: Learn how to create a fully private EKS cluster with private API endpoint access using OpenTofu, requiring VPC endpoints for AWS service connectivity without internet access.

## Introduction

A private EKS cluster has only a private endpoint for the Kubernetes API server, meaning all communication to and from the cluster stays within your VPC. This is required for high-security environments and eliminates internet exposure of the control plane.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with EKS, VPC, and EC2 permissions
- kubectl with the ability to connect via VPC (bastion, VPN, or Direct Connect)

## Step 1: Create a Private EKS Cluster

```hcl
resource "aws_eks_cluster" "private" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids = var.private_subnet_ids

    # Disable public endpoint - API only accessible within VPC
    endpoint_public_access  = false
    endpoint_private_access = true
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator"]

  tags = {
    Name    = var.cluster_name
    Private = "true"
  }
}
```

## Step 2: Create Required VPC Endpoints

```hcl
# Security group for VPC endpoints

resource "aws_security_group" "vpc_endpoints" {
  name        = "${var.cluster_name}-vpc-endpoints-sg"
  description = "Security group for VPC endpoints"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }
}

# S3 Gateway endpoint (free, no security group needed)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = var.vpc_id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = var.private_route_table_ids
}

# ECR API endpoint for pulling images
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# EC2 endpoint for node provisioning
resource "aws_vpc_endpoint" "ec2" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.ec2"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# CloudWatch Logs endpoint for log shipping
resource "aws_vpc_endpoint" "cloudwatch_logs" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.logs"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# STS endpoint for IAM role assumption
resource "aws_vpc_endpoint" "sts" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.sts"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}
```

## Step 3: kubectl Access via EC2 Instance Connect Endpoint

```hcl
# Bastion for kubectl access when Direct Connect or VPN is not available
resource "aws_instance" "bastion" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = var.private_subnet_ids[0]

  # No public IP - access via Instance Connect Endpoint
  associate_public_ip_address = false

  iam_instance_profile = aws_iam_instance_profile.bastion.name

  user_data = base64encode(<<-EOF
    #!/bin/bash
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    aws eks update-kubeconfig --region ${var.region} --name ${var.cluster_name}
  EOF
  )

  tags = { Name = "eks-private-bastion" }
}
```

## Step 4: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

A private EKS cluster with VPC endpoints provides maximum security by keeping all cluster traffic within your AWS network. The minimum required VPC endpoints are: ECR API, ECR DKR, S3 (gateway), EC2, STS, and CloudWatch Logs. Access kubectl via a bastion, AWS Systems Manager Session Manager, Direct Connect, or VPN to administer the private cluster.
