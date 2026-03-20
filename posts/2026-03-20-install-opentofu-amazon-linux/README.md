# How to Install OpenTofu on Amazon Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Amazon Linux, AWS, Installation, Infrastructure as Code, DevOps

Description: A guide to installing OpenTofu on Amazon Linux 2 and Amazon Linux 2023 for AWS infrastructure management.

## Introduction

Amazon Linux is AWS's own Linux distribution optimized for running on EC2 instances. Installing OpenTofu on Amazon Linux is particularly valuable when you want to manage AWS infrastructure from within EC2 instances or build CI/CD pipelines using AWS CodeBuild.

## Prerequisites

- Amazon Linux 2 or Amazon Linux 2023
- `sudo` or root access
- Internet access (or VPC endpoints for S3/GitHub)

## Method 1: Install via YUM/DNF Repository

### For Amazon Linux 2023 (uses dnf)

```bash
# Add the OpenTofu repository
cat <<EOF | sudo tee /etc/yum.repos.d/opentofu.repo
[opentofu]
name=opentofu
baseurl=https://packages.opentofu.org/opentofu/tofu/rpm_any/rpm_any/\$basearch
repo_gpgcheck=1
gpgcheck=1
enabled=1
gpgkey=https://get.opentofu.org/opentofu.gpg
       https://packages.opentofu.org/opentofu/tofu/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
EOF

# Install OpenTofu
sudo dnf install -y tofu
```

### For Amazon Linux 2 (uses yum)

```bash
# Add the repository
cat <<EOF | sudo tee /etc/yum.repos.d/opentofu.repo
[opentofu]
name=opentofu
baseurl=https://packages.opentofu.org/opentofu/tofu/rpm_any/rpm_any/\$basearch
repo_gpgcheck=1
gpgcheck=1
enabled=1
gpgkey=https://get.opentofu.org/opentofu.gpg
       https://packages.opentofu.org/opentofu/tofu/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
EOF

# Import GPG key
sudo rpm --import https://get.opentofu.org/opentofu.gpg

# Install OpenTofu
sudo yum install -y tofu
```

## Method 2: Install from Binary

```bash
TOFU_VERSION="1.9.0"

# Install unzip if needed
sudo yum install -y unzip  # or dnf for AL2023

# Download the binary
curl -LO "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_linux_amd64.zip"

# Extract and install
unzip "tofu_${TOFU_VERSION}_linux_amd64.zip"
sudo mv tofu /usr/local/bin/
sudo chmod +x /usr/local/bin/tofu

# Clean up
rm "tofu_${TOFU_VERSION}_linux_amd64.zip"
```

## Using OpenTofu with AWS on EC2

When running on EC2 with an IAM instance role, OpenTofu can use the instance metadata service to authenticate with AWS:

```hcl
# main.tf - Managing AWS resources from EC2
terraform {
  required_version = ">= 1.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Use S3 for remote state when running on EC2
  backend "s3" {
    bucket = "my-tofu-state-bucket"
    key    = "infra/terraform.tfstate"
    region = "us-east-1"
  }
}

# Provider will use the EC2 instance role automatically
provider "aws" {
  region = "us-east-1"
}

# Example: Create an S3 bucket
resource "aws_s3_bucket" "example" {
  bucket = "my-example-bucket-from-ec2"

  tags = {
    ManagedBy = "OpenTofu"
    Environment = "dev"
  }
}
```

## Setting Up for AWS CodeBuild

Use OpenTofu in AWS CodeBuild pipelines:

```yaml
# buildspec.yml for AWS CodeBuild
version: 0.2

phases:
  install:
    commands:
      - TOFU_VERSION="1.9.0"
      - curl -LO "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_linux_amd64.zip"
      - unzip "tofu_${TOFU_VERSION}_linux_amd64.zip"
      - mv tofu /usr/local/bin/
      - tofu version

  build:
    commands:
      - cd infrastructure/
      - tofu init
      - tofu plan -out=tfplan
      - tofu apply -auto-approve tfplan
```

## Verifying the Installation

```bash
# Check the version
tofu version

# Test with AWS provider (requires IAM permissions)
tofu providers
```

## Conclusion

Installing OpenTofu on Amazon Linux enables powerful infrastructure-as-code workflows directly within the AWS ecosystem. Whether you are using EC2 instances, CodeBuild, or other compute services, OpenTofu integrates seamlessly with AWS IAM for authentication, making it an excellent choice for managing your cloud infrastructure.
