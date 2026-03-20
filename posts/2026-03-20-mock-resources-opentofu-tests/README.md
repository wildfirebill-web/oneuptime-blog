# How to Mock Resources in OpenTofu Tests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Testing, IaC, DevOps, Terraform

Description: Learn how to use mock_resource blocks in OpenTofu tests to define fake resource behavior and return values for unit testing without real infrastructure.

## Introduction

When using `mock_provider` in OpenTofu tests, `mock_resource` blocks let you define the attributes that a mocked resource returns after "creation". Without explicit mock_resource defaults, OpenTofu generates random values. Defining them explicitly ensures your assertions can check for specific values.

## Basic mock_resource Syntax

```hcl
mock_provider "aws" {
  mock_resource "aws_instance" {
    defaults = {
      id             = "i-0123456789abcdef0"
      public_ip      = "54.1.2.3"
      private_ip     = "10.0.1.10"
      instance_state = "running"
      arn            = "arn:aws:ec2:us-east-1:123456789012:instance/i-0123456789abcdef0"
    }
  }
}
```

## Common Resource Mocks

### EC2 Instance

```hcl
mock_provider "aws" {
  mock_resource "aws_instance" {
    defaults = {
      id                   = "i-0abc123def456789"
      public_ip            = "54.10.20.30"
      private_ip           = "10.0.1.100"
      public_dns           = "ec2-54-10-20-30.compute-1.amazonaws.com"
      private_dns          = "ip-10-0-1-100.ec2.internal"
      instance_state       = "running"
      availability_zone    = "us-east-1a"
      subnet_id            = "subnet-0abc123"
      vpc_security_group_ids = ["sg-0abc123"]
      arn                  = "arn:aws:ec2:us-east-1:123456789012:instance/i-0abc123def456789"
    }
  }
}
```

### S3 Bucket

```hcl
mock_provider "aws" {
  mock_resource "aws_s3_bucket" {
    defaults = {
      id                  = "my-test-bucket"
      bucket              = "my-test-bucket"
      bucket_domain_name  = "my-test-bucket.s3.amazonaws.com"
      bucket_regional_domain_name = "my-test-bucket.s3.us-east-1.amazonaws.com"
      arn                 = "arn:aws:s3:::my-test-bucket"
      region              = "us-east-1"
    }
  }
}
```

### RDS Instance

```hcl
mock_provider "aws" {
  mock_resource "aws_db_instance" {
    defaults = {
      id                  = "mock-db-identifier"
      address             = "mock-db.abc123def456.us-east-1.rds.amazonaws.com"
      endpoint            = "mock-db.abc123def456.us-east-1.rds.amazonaws.com:5432"
      port                = 5432
      status              = "available"
      multi_az            = true
      engine              = "postgres"
      engine_version      = "14.8"
      arn                 = "arn:aws:rds:us-east-1:123456789012:db:mock-db"
    }
  }
}
```

### IAM Role

```hcl
mock_provider "aws" {
  mock_resource "aws_iam_role" {
    defaults = {
      id          = "mock-app-role"
      name        = "mock-app-role"
      arn         = "arn:aws:iam::123456789012:role/mock-app-role"
      unique_id   = "AROAMOCKTEST123456789"
    }
  }
}
```

### VPC and Subnets

```hcl
mock_provider "aws" {
  mock_resource "aws_vpc" {
    defaults = {
      id         = "vpc-0abc123def456789a"
      cidr_block = "10.0.0.0/16"
      arn        = "arn:aws:ec2:us-east-1:123456789012:vpc/vpc-0abc123"
    }
  }

  mock_resource "aws_subnet" {
    defaults = {
      id                = "subnet-0abc123def456789a"
      vpc_id            = "vpc-0abc123def456789a"
      availability_zone = "us-east-1a"
      cidr_block        = "10.0.1.0/24"
      arn               = "arn:aws:ec2:us-east-1:123456789012:subnet/subnet-0abc123"
    }
  }
}
```

## Using Mocks in Test Assertions

```hcl
mock_provider "aws" {
  mock_resource "aws_instance" {
    defaults = {
      id        = "i-mocked"
      public_ip = "1.2.3.4"
      arn       = "arn:aws:ec2:us-east-1:123456789012:instance/i-mocked"
    }
  }
}

run "instance_outputs_are_populated" {
  command = plan

  assert {
    # The module outputs the instance ARN - verify it's set
    condition     = output.instance_arn == "arn:aws:ec2:us-east-1:123456789012:instance/i-mocked"
    error_message = "Instance ARN output not set correctly from mock"
  }

  assert {
    condition     = output.instance_public_ip == "1.2.3.4"
    error_message = "Instance public IP output not matching mock value"
  }
}
```

## Default Behavior Without mock_resource

When you use `mock_provider` without `mock_resource`, OpenTofu auto-generates values:

```hcl
# Without mock_resource - all attributes get auto-generated values

mock_provider "aws" {}

run "basic_plan_test" {
  command = plan

  assert {
    # Can only check config-derived values, not provider-assigned ones
    condition     = aws_instance.web.instance_type == var.instance_type
    error_message = "Instance type should match variable"
  }
}
```

## Conclusion

`mock_resource` blocks give you control over what attribute values a mocked resource returns. Define explicit defaults to make your assertions deterministic and reliable. Include the key computed attributes (like `id`, `arn`, `address`) that your module uses in outputs or other resources. Without explicit mocks, auto-generated values can make assertions on provider-computed attributes difficult.
