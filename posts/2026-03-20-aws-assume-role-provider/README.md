# How to Use AWS Assume Role in the AWS Provider

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, AWS, Terraform, IaC, DevOps, IAM

Description: Learn how to configure the AWS provider in OpenTofu to assume an IAM role, enabling cross-account deployments and least-privilege access patterns.

## Introduction

The AWS provider's `assume_role` configuration allows OpenTofu to assume a different IAM role before making API calls. This is the standard pattern for cross-account deployments (deploying to a target account from a CI/CD account), for enforcing least-privilege access, and for using OIDC federation with GitHub Actions or other CI systems.

## Basic Assume Role Configuration

```hcl
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/opentofu-deploy-role"
    session_name = "opentofu-session"
  }
}
```

## Assume Role with External ID

The external ID adds a security condition (useful for third-party tools):

```hcl
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::TARGET_ACCOUNT_ID:role/opentofu-role"
    session_name = "opentofu-deploy"
    external_id  = var.external_id  # Must match the IAM role trust policy
  }
}
```

## Cross-Account Deployment Pattern

The common pattern: CI/CD runs in account A, deploys to account B.

```hcl
# Account A: CI/CD account (where OpenTofu runs)

# Account B: Target account (where resources are created)

provider "aws" {
  region = var.region

  assume_role {
    # Role in account B that allows deployment
    role_arn     = "arn:aws:iam::${var.target_account_id}:role/opentofu-deploy"
    session_name = "opentofu-${var.environment}"
    duration_seconds = 3600  # 1-hour session
  }
}
```

The IAM role in account B needs a trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::CI_CD_ACCOUNT_ID:role/github-actions-role"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

## Multi-Account Configuration with Aliases

Deploy to multiple accounts in one configuration:

```hcl
# Deploy to shared services account
provider "aws" {
  alias  = "shared"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::${var.shared_account_id}:role/opentofu-shared-role"
  }
}

# Deploy to workload account
provider "aws" {
  alias  = "workload"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::${var.workload_account_id}:role/opentofu-workload-role"
  }
}

# Create resources in shared account
resource "aws_route53_zone" "main" {
  provider = aws.shared
  name     = var.domain_name
}

# Create resources in workload account
resource "aws_instance" "app" {
  provider      = aws.workload
  ami           = var.ami_id
  instance_type = "t3.micro"
}
```

## OIDC Authentication (GitHub Actions)

Use OIDC tokens instead of access keys:

```hcl
# The AWS provider automatically uses OIDC when configured
provider "aws" {
  region = "us-east-1"

  assume_role_with_web_identity {
    role_arn                = "arn:aws:iam::123456789012:role/github-actions-role"
    web_identity_token_file = "/tmp/oidc-token"  # Set by GitHub Actions
    session_name            = "github-actions-${var.environment}"
  }
}
```

```yaml
# GitHub Actions workflow
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
    aws-region: us-east-1

- name: OpenTofu Apply
  run: tofu apply -auto-approve
```

## IAM Role for OpenTofu

Create the deployment role with OpenTofu itself:

```hcl
# In your shared services / bootstrap configuration
resource "aws_iam_role" "opentofu_deploy" {
  name = "opentofu-deploy-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        AWS = "arn:aws:iam::${var.cicd_account_id}:role/github-actions"
      }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "opentofu_deploy" {
  role       = aws_iam_role.opentofu_deploy.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
  # In production, use a more restrictive custom policy
}
```

## Session Tags for Auditing

Add session tags to identify individual deploys:

```hcl
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/opentofu-role"
    session_name = "opentofu-deploy"

    tags = {
      DeployedBy  = "github-actions"
      Environment = var.environment
      Repository  = "my-org/infrastructure"
    }

    # Require these tags to be present on assumed sessions
    transitive_tag_keys = ["DeployedBy", "Environment"]
  }
}
```

## Conclusion

`assume_role` in the AWS provider enables clean cross-account deployments and least-privilege access patterns. Use it with `session_name` for audit trail clarity, `duration_seconds` to limit session lifetime, and OIDC for keyless CI/CD authentication. For multi-account organizations, combine assume_role with provider aliases to manage multiple target accounts from a single OpenTofu configuration. Always prefer OIDC federation over long-lived access keys for CI/CD workflows.
