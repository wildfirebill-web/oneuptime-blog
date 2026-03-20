# How to Define a Resource Block in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Resources, HCL, Infrastructure as Code, DevOps

Description: A guide to defining and configuring resource blocks in OpenTofu to provision and manage infrastructure objects.

## Introduction

Resource blocks are the fundamental building blocks of OpenTofu configurations. Each resource block describes one or more infrastructure objects, such as virtual machines, networks, databases, or DNS records. Understanding resource block syntax and structure is essential for writing effective OpenTofu configurations.

## Resource Block Syntax

```hcl
# Syntax:
# resource "<RESOURCE_TYPE>" "<LOCAL_NAME>" {
#   argument = value
# }

# Example: AWS EC2 instance
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

# Example: Google Cloud compute instance
resource "google_compute_instance" "app" {
  name         = "app-server"
  machine_type = "e2-medium"
  zone         = "us-central1-a"
}
```

## Resource Block Components

```hcl
# 1. Resource Type: identifies the provider and resource type
# Format: <PROVIDER>_<RESOURCE_TYPE>
# aws_instance = AWS provider, instance resource type
# azurerm_virtual_machine = Azure RM provider, virtual machine

# 2. Local Name: a unique identifier within the module
# Used to reference this resource elsewhere: aws_instance.web_server

resource "aws_instance" "web_server" {
  # 3. Arguments: configure the resource
  ami           = "ami-0c55b159cbfafe1f0"  # Required argument
  instance_type = "t3.micro"               # Required argument

  # Optional arguments
  key_name               = "my-keypair"
  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = aws_subnet.public.id

  # Nested blocks for complex configuration
  root_block_device {
    volume_size           = 20
    volume_type           = "gp3"
    delete_on_termination = true
  }

  # Meta-arguments: special arguments for all resources
  count      = 2                              # Create 2 instances
  depends_on = [aws_security_group.web]       # Explicit dependency
  provider   = aws.west                       # Specific provider alias

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false
    ignore_changes        = [ami]
  }

  # Tags
  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}
```

## Referencing Resource Attributes

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  # Reference another resource's attribute
  vpc_id     = aws_vpc.main.id      # Syntax: <TYPE>.<NAME>.<ATTRIBUTE>
  cidr_block = "10.0.1.0/24"
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id  # Same resource referenced again
}

output "subnet_id" {
  value = aws_subnet.public.id  # Reference in output
}
```

## Multiple Resources of the Same Type

```hcl
# Two different S3 buckets
resource "aws_s3_bucket" "logs" {
  bucket = "my-app-logs"
}

resource "aws_s3_bucket" "assets" {
  bucket = "my-app-assets"
}

# Reference by local name
output "log_bucket_arn" {
  value = aws_s3_bucket.logs.arn
}

output "assets_bucket_arn" {
  value = aws_s3_bucket.assets.arn
}
```

## Complete Resource Example

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name      = "main-vpc"
    ManagedBy = "OpenTofu"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id  # Reference vpc
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = "us-east-1a"

  tags = {
    Name = "public-subnet"
  }
}

output "vpc_id" {
  value = aws_vpc.main.id
}
```

## Conclusion

Resource blocks are where OpenTofu configurations define the actual infrastructure to be created. Understanding the structure — resource type, local name, arguments, nested blocks, and meta-arguments — enables you to configure any resource supported by any provider. The reference syntax `<type>.<name>.<attribute>` creates the dependency graph that OpenTofu uses to determine creation order and parallelism.
