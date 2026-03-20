# How to Create AWS Batch Compute Environments with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Batch, Compute, HPC, Infrastructure as Code

Description: Learn how to create AWS Batch managed and unmanaged compute environments with EC2 and Fargate launch types using OpenTofu.

## Introduction

AWS Batch manages the provisioning of compute resources for batch processing workloads. Compute Environments define the instance types, VPC, and capacity limits available to your jobs. OpenTofu manages these as code for reproducible batch infrastructure.

## IAM Roles

```hcl
# Service role for Batch to manage EC2 instances
resource "aws_iam_role" "batch_service" {
  name = "aws-batch-service-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "batch.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "batch_service" {
  role       = aws_iam_role.batch_service.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSBatchServiceRole"
}

# Instance profile for EC2 instances in the compute environment
resource "aws_iam_instance_profile" "ecs_instance" {
  name = "ecsInstanceRole"
  role = aws_iam_role.ecs_instance.name
}

resource "aws_iam_role" "ecs_instance" {
  name = "ecsInstanceRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_instance" {
  role       = aws_iam_role.ecs_instance.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role"
}
```

## Managed EC2 Compute Environment

```hcl
resource "aws_batch_compute_environment" "ec2" {
  compute_environment_name = "${var.app_name}-ec2-${var.environment}"
  type                     = "MANAGED"
  service_role             = aws_iam_role.batch_service.arn
  state                    = "ENABLED"

  compute_resources {
    type      = "EC2"
    min_vcpus = 0
    max_vcpus = 256
    desired_vcpus = 0

    instance_type = ["m5.large", "m5.xlarge", "m5.2xlarge", "c5.large", "c5.xlarge"]

    subnets            = var.private_subnet_ids
    security_group_ids = [aws_security_group.batch.id]
    instance_role      = aws_iam_instance_profile.ecs_instance.arn

    tags = {
      Environment = var.environment
      ManagedBy   = "opentofu"
    }
  }

  tags = {
    Environment = var.environment
    ManagedBy   = "opentofu"
  }
}
```

## Spot Compute Environment

```hcl
resource "aws_batch_compute_environment" "spot" {
  compute_environment_name = "${var.app_name}-spot-${var.environment}"
  type                     = "MANAGED"
  service_role             = aws_iam_role.batch_service.arn

  compute_resources {
    type                = "SPOT"
    bid_percentage      = 60  # bid up to 60% of on-demand price
    spot_iam_fleet_role = aws_iam_role.spot_fleet.arn

    min_vcpus = 0
    max_vcpus = 512

    instance_type      = ["optimal"]  # let AWS choose best instance families
    subnets            = var.private_subnet_ids
    security_group_ids = [aws_security_group.batch.id]
    instance_role      = aws_iam_instance_profile.ecs_instance.arn
  }
}
```

## Fargate Compute Environment

```hcl
resource "aws_batch_compute_environment" "fargate" {
  compute_environment_name = "${var.app_name}-fargate-${var.environment}"
  type                     = "MANAGED"
  service_role             = aws_iam_role.batch_service.arn

  compute_resources {
    type               = "FARGATE"
    max_vcpus          = 256
    subnets            = var.private_subnet_ids
    security_group_ids = [aws_security_group.batch.id]
  }
}
```

## Security Group

```hcl
resource "aws_security_group" "batch" {
  name   = "${var.app_name}-batch"
  vpc_id = var.vpc_id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

AWS Batch Compute Environments define the compute capacity available for batch jobs. OpenTofu manages EC2, Spot, and Fargate compute environments along with the required IAM roles and security groups, creating a fully automated batch processing foundation.
