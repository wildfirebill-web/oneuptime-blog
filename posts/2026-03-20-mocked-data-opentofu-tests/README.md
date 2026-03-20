# How to Use Mocked Data in OpenTofu Tests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Mocked Data, Mock Provider, Test Framework

Description: Learn how to use OpenTofu's mock_provider and mock data sources to create realistic test scenarios without cloud credentials, enabling fast and deterministic module testing.

## Introduction

OpenTofu's test framework supports mocking both resource creation (`mock_resource`) and data source lookups (`mock_data`). Mocking data sources is essential for testing modules that use `data` blocks to look up existing infrastructure — without mocks, these data sources require real cloud access even in unit tests.

## Mocking Data Sources

```hcl
# tests/with_mocked_data.tftest.hcl

mock_provider "aws" {
  # Mock a data source that looks up an AMI
  mock_data "aws_ami" {
    defaults = {
      id            = "ami-mock-ubuntu22"
      name          = "ubuntu/images/hvm-ssd/ubuntu-22.04-amd64-server-20240101"
      owner_id      = "099720109477"
      architecture  = "x86_64"
      root_device_type = "ebs"
      virtualization_type = "hvm"
      state         = "available"
    }
  }

  # Mock a data source that looks up the current AWS region
  mock_data "aws_region" {
    defaults = {
      name        = "us-east-1"
      description = "US East (N. Virginia)"
    }
  }

  # Mock caller identity
  mock_data "aws_caller_identity" {
    defaults = {
      account_id = "123456789012"
      arn        = "arn:aws:iam::123456789012:user/test"
      user_id    = "AIDA123456789012"
    }
  }
}

run "uses_correct_ami" {
  command = plan

  variables {
    instance_type = "t3.medium"
    environment   = "test"
  }

  assert {
    condition     = aws_instance.main.ami == "ami-mock-ubuntu22"
    error_message = "Instance should use the AMI returned by the data source"
  }
}
```

## Mocking Resources with Computed Values

```hcl
mock_provider "aws" {
  # Mock resources that have computed attributes needed by other resources
  mock_resource "aws_vpc" {
    defaults = {
      id                     = "vpc-0123456789abcdef0"
      cidr_block             = "10.0.0.0/16"
      default_route_table_id = "rtb-0123456789abcdef0"
      main_route_table_id    = "rtb-0123456789abcdef1"
      default_network_acl_id = "acl-0123456789abcdef0"
      enable_dns_hostnames   = true
      enable_dns_support     = true
    }
  }

  mock_resource "aws_subnet" {
    defaults = {
      id                = "subnet-0123456789abcdef0"
      vpc_id            = "vpc-0123456789abcdef0"
      cidr_block        = "10.0.1.0/24"
      availability_zone = "us-east-1a"
      map_public_ip_on_launch = false
    }
  }

  mock_resource "aws_security_group" {
    defaults = {
      id   = "sg-0123456789abcdef0"
      name = "test-sg"
      vpc_id = "vpc-0123456789abcdef0"
    }
  }
}
```

## Mocking Provider-Level Functions

```hcl
mock_provider "aws" {
  # Set up data source returns for SSM parameter lookups
  mock_data "aws_ssm_parameter" {
    defaults = {
      value   = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64"
      version = "1"
      type    = "String"
      arn     = "arn:aws:ssm:us-east-1::parameter/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64"
    }
  }

  # Mock the data source for availability zones
  mock_data "aws_availability_zones" {
    defaults = {
      names = ["us-east-1a", "us-east-1b", "us-east-1c"]
      state = "available"
    }
  }
}
```

## Scenario-Based Mocking

```hcl
# Test scenario: multi-region deployment
mock_provider "aws" {
  alias = "us_east"
  mock_data "aws_caller_identity" {
    defaults = {
      account_id = "111111111111"
    }
  }
}

mock_provider "aws" {
  alias = "us_west"
  mock_data "aws_caller_identity" {
    defaults = {
      account_id = "222222222222"
    }
  }
}

run "multi_region_deployment" {
  command = plan

  variables {
    primary_region   = "us-east-1"
    secondary_region = "us-west-2"
  }

  assert {
    condition     = aws_s3_bucket.primary.bucket != aws_s3_bucket.secondary.bucket
    error_message = "Primary and secondary buckets must have different names"
  }
}
```

## Mocking HTTP Data Sources

```hcl
# For modules that use the http provider to fetch external data
mock_provider "http" {
  mock_data "http" {
    defaults = {
      response_body    = "{\"ip\":\"203.0.113.1\"}"
      status_code      = 200
      response_headers = { "Content-Type" = "application/json" }
    }
  }
}
```

## Best Practices for Mock Data

```hcl
# Use realistic mock values that match real data formats
mock_provider "aws" {
  mock_resource "aws_lb" {
    defaults = {
      id       = "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/test/1234567890abcdef"
      dns_name = "test-1234567890.us-east-1.elb.amazonaws.com"
      zone_id  = "Z35SXDOTRQ7X7K"  # Real ALB hosted zone ID for us-east-1
      arn      = "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/test/1234567890abcdef"
    }
  }
}
```

## Conclusion

Mock data in OpenTofu tests enables testing the full module logic including data source lookups without cloud credentials. Use realistic mock values that match the format of real AWS responses — this catches bugs where your code assumes specific data formats. The key insight is that `mock_data` lets you test code paths that depend on external data sources (AMI lookups, SSM parameters, availability zones) while keeping tests fast and offline.
