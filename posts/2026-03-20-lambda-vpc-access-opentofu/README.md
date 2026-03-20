# How to Set Up Lambda with VPC Access Using OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Lambda, VPC, Networking, Serverless, Infrastructure as Code

Description: Learn how to configure Lambda functions with VPC access using OpenTofu to enable connectivity to RDS, ElastiCache, and other private resources in your VPC.

## Introduction

By default, Lambda functions run in an AWS-managed VPC with internet access but cannot reach private resources in your VPC. Enabling VPC access places Lambda in your private subnets, allowing connectivity to databases, cache clusters, and other private services.

## Prerequisites

- OpenTofu v1.6+
- An existing VPC with private subnets
- AWS credentials with Lambda, EC2, and IAM permissions

## Step 1: Create the Lambda Execution Role with VPC Policy

```hcl
resource "aws_iam_role" "lambda_vpc" {
  name = "lambda-vpc-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

# This policy grants permission to create and manage ENIs in the VPC

resource "aws_iam_role_policy_attachment" "lambda_vpc_policy" {
  role       = aws_iam_role.lambda_vpc.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_vpc.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

## Step 2: Create the Lambda Security Group

```hcl
# Security group controlling Lambda's network access
resource "aws_security_group" "lambda" {
  name        = "lambda-vpc-sg"
  description = "Security group for Lambda VPC functions"
  vpc_id      = var.vpc_id

  # Allow outbound to RDS on port 5432
  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.rds.id]
    description     = "PostgreSQL access"
  }

  # Allow outbound to Redis on port 6379
  egress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.redis.id]
    description     = "Redis access"
  }

  # Allow HTTPS outbound via NAT gateway for external APIs
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS outbound"
  }

  tags = { Name = "lambda-vpc-sg" }
}
```

## Step 3: Configure the Lambda Function with VPC Settings

```hcl
resource "aws_lambda_function" "db_processor" {
  function_name    = "db-processor"
  role             = aws_iam_role.lambda_vpc.arn
  handler          = "index.handler"
  runtime          = "python3.12"
  filename         = data.archive_file.zip.output_path
  source_code_hash = data.archive_file.zip.output_base64sha256
  memory_size      = 512
  timeout          = 60

  # VPC configuration - place Lambda in private subnets
  vpc_config {
    subnet_ids         = var.private_subnet_ids    # Private subnets across multiple AZs
    security_group_ids = [aws_security_group.lambda.id]
  }

  environment {
    variables = {
      DB_HOST     = var.rds_endpoint
      DB_PORT     = "5432"
      DB_NAME     = var.database_name
      REDIS_HOST  = var.redis_endpoint
    }
  }

  tags = { Name = "db-processor" }
}
```

## Step 4: Grant RDS and Redis Security Groups Access from Lambda

```hcl
# Allow Lambda to connect to RDS
resource "aws_security_group_rule" "lambda_to_rds" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.rds.id
  source_security_group_id = aws_security_group.lambda.id
  description              = "Allow Lambda to connect to RDS"
}

# Allow Lambda to connect to ElastiCache Redis
resource "aws_security_group_rule" "lambda_to_redis" {
  type                     = "ingress"
  from_port                = 6379
  to_port                  = 6379
  protocol                 = "tcp"
  security_group_id        = aws_security_group.redis.id
  source_security_group_id = aws_security_group.lambda.id
  description              = "Allow Lambda to connect to Redis"
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

VPC-enabled Lambda functions can access private resources like RDS, ElastiCache, and internal APIs. Place Lambda in private subnets with a NAT Gateway for outbound internet access. Be aware that VPC attachment adds 1-10 seconds to cold start time unless using provisioned concurrency, and Lambda may exhaust subnet IP addresses under high concurrency-use subnets with /24 or larger CIDR blocks.
