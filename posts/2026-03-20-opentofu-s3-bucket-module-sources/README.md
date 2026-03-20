# How to Use S3 Bucket Module Sources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Modules, AWS

Description: Learn how to use S3 bucket URLs as module sources in OpenTofu to distribute private modules securely within AWS environments.

## Introduction

OpenTofu supports fetching modules directly from Amazon S3 buckets using the `s3::` prefix. This is ideal for organizations that store versioned module archives in S3 and need fine-grained IAM-based access control without running a module registry.

## Syntax

```hcl
module "name" {
  source = "s3::https://s3.amazonaws.com/BUCKET/path/to/module.zip"
}
```

The `s3::` prefix tells OpenTofu to use the S3 protocol handler. The URL follows the standard S3 HTTPS endpoint format.

## Basic S3 Module Source

```hcl
# Fetch a versioned module archive from S3

module "vpc" {
  source = "s3::https://s3.amazonaws.com/acme-terraform-modules/vpc/v2.1.0.zip"

  name       = "production"
  cidr_block = "10.0.0.0/16"
}
```

## Using Regional Endpoints

For buckets in specific regions, use the regional endpoint to avoid redirect latency:

```hcl
module "eks" {
  source = "s3::https://s3.us-west-2.amazonaws.com/acme-modules-us-west-2/eks/v1.5.0.zip"

  cluster_name    = "prod-cluster"
  cluster_version = "1.28"
}
```

## Pointing to a Subdirectory

Use the `//` separator to reference a subdirectory within the archive:

```hcl
module "security_groups" {
  source = "s3::https://s3.amazonaws.com/acme-modules/aws-networking-v3.zip//security-groups"

  vpc_id      = var.vpc_id
  environment = var.environment
}
```

## Authentication

OpenTofu uses the standard AWS credential chain - the same mechanism used by the AWS CLI and Terraform AWS provider:

```bash
# Option 1: Environment variables
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

# Option 2: AWS profile
export AWS_PROFILE="terraform-modules"

# Option 3: Instance profile / IRSA (no credentials needed when running on EC2/EKS)
```

## IAM Policy for Module Access

Restrict access to module archives using S3 bucket policies:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::acme-terraform-modules/*",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/terraform-deployer"
      }
    }
  ]
}
```

## Organizing Modules in S3

A consistent naming convention makes modules easy to discover:

```text
s3://acme-terraform-modules/
├── vpc/
│   ├── v1.0.0.zip
│   ├── v1.1.0.zip
│   └── v2.0.0.zip
├── eks/
│   ├── v1.0.0.zip
│   └── v1.5.0.zip
└── rds/
    └── v1.0.0.zip
```

```hcl
# Reference specific versions by embedding them in the path
module "rds" {
  source = "s3::https://s3.amazonaws.com/acme-terraform-modules/rds/v1.0.0.zip"

  engine         = "postgres"
  instance_class = "db.t3.medium"
}
```

## Uploading Modules to S3

```bash
# Package and upload a module version
zip -r vpc-v2.1.0.zip ./modules/vpc/
aws s3 cp vpc-v2.1.0.zip s3://acme-terraform-modules/vpc/v2.1.0.zip

# Verify the upload
aws s3 ls s3://acme-terraform-modules/vpc/
```

## Important Notes

- OpenTofu re-downloads the archive on every `tofu init`. Embed the version in the S3 key path for stability.
- S3 module sources work with all AWS authentication methods: static credentials, instance profiles, IRSA, and SSO.
- Use S3 versioning or key-based versioning for reproducible builds.
- Bucket names must be globally unique; use a company-specific prefix.

## Conclusion

S3 bucket module sources are the natural choice for AWS-native teams. They provide IAM-enforced access control, high availability, and integration with existing AWS infrastructure. Store versioned archives with clear key conventions for reliable, reproducible module consumption across your organization.
