# How to Configure AWS ECR Access over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS ECR, IPv6, Container Registry, AWS, Docker, VPC, Networking

Description: Configure AWS Elastic Container Registry access over IPv6 using dual-stack VPC endpoints, ECR public IPv6 endpoints, and proper IAM and network configuration.

---

AWS ECR supports IPv6 through dual-stack endpoints. Accessing ECR from IPv6-capable infrastructure requires configuring VPC endpoints with dual-stack support or using the public ECR endpoints that support IPv6.

## ECR IPv6 Support Overview

AWS ECR provides IPv6 support through:
1. **Public ECR endpoints** — `public.ecr.aws` has IPv6 support.
2. **Private VPC endpoints** — ECR VPC endpoints support dual-stack (IPv4/IPv6).
3. **Direct internet access** — ECR's regional endpoints support IPv6.

## Checking ECR IPv6 Availability

```bash
# Check if ECR regional endpoint resolves to IPv6
dig AAAA 123456789.dkr.ecr.us-east-1.amazonaws.com +short

# Check ECR API endpoint
dig AAAA ecr.us-east-1.amazonaws.com +short

# Check ECR Public Gallery
dig AAAA public.ecr.aws +short

# Test connectivity
curl -6 https://public.ecr.aws/v2/ 2>&1 | head -5
```

## Configuring AWS CLI for IPv6

```bash
# Install/configure AWS CLI
aws configure

# Test ECR authentication over IPv6
aws ecr get-login-password \
  --region us-east-1 | \
  docker login \
  --username AWS \
  --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# List repositories
aws ecr describe-repositories --region us-east-1
```

## Setting Up ECR VPC Endpoint with Dual-Stack

For EC2 instances in a VPC with IPv6:

```bash
# Create ECR API VPC endpoint with dual-stack
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.ecr.api \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-12345678 \
  --ip-address-type dualstack \
  --security-group-ids sg-12345678 \
  --region us-east-1

# Create ECR Docker VPC endpoint with dual-stack
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-12345678 \
  --ip-address-type dualstack \
  --security-group-ids sg-12345678 \
  --region us-east-1
```

## IPv6 VPC Configuration for ECR Access

Your VPC and EC2 instances need IPv6 properly configured:

```bash
# Check if EC2 instance has IPv6 address
ip -6 addr show | grep "scope global"

# Check the VPC has IPv6 CIDR assigned
aws ec2 describe-vpcs \
  --vpc-ids vpc-12345678 \
  --query 'Vpcs[].Ipv6CidrBlockAssociationSet'

# Ensure IPv6 route exists
ip -6 route show default

# Test internet connectivity over IPv6 from EC2
curl -6 https://ifconfig.me/ip
```

## Pushing and Pulling Images over IPv6

```bash
# Authenticate to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Build and push image
docker build -t myapp .
docker tag myapp:latest \
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push \
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# Pull image
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# Verify IPv6 was used (watch network traffic)
sudo tcpdump -i eth0 -n ip6 and host ecr.us-east-1.amazonaws.com
```

## ECR Public Registry over IPv6

ECR Public Gallery supports IPv6:

```bash
# Pull public images over IPv6
docker pull public.ecr.aws/nginx/nginx:latest

# Authenticate to ECR Public (optional for pulling public images)
aws ecr-public get-login-password \
  --region us-east-1 | \
  docker login \
  --username AWS \
  --password-stdin \
  public.ecr.aws

# Push to ECR Public
docker tag myapp:latest public.ecr.aws/myalias/myapp:latest
docker push public.ecr.aws/myalias/myapp:latest
```

## IAM Policy for ECR IPv6 Access

IAM policies are not IP-version specific, but ensure the right permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    }
  ]
}
```

AWS ECR's dual-stack endpoint support enables container image management over IPv6 from modern AWS infrastructure, fully compatible with EKS clusters and EC2 instances running in IPv6-enabled VPCs.
