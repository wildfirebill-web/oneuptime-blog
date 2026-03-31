# How to Split State by Environment in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, State Management, Environment, S3 Backend, Best Practice

Description: Learn how to split OpenTofu state files by environment to limit blast radius, enable independent deployments, and prevent cross-environment state pollution.

## Introduction

Storing all environments in a single state file means a misconfigured plan for dev could affect production resources. Splitting state by environment isolates risk and enables independent deployment cadences per environment.

## Directory-Based State Separation

The most common approach uses separate directories with environment-specific backend configurations.

```text
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── backend.tf
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── backend.tf
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       └── backend.tf
└── modules/
    ├── vpc/
    └── app-stack/
```

## Backend Configuration Per Environment

```hcl
# environments/dev/backend.tf

terraform {
  backend "s3" {
    bucket         = "my-company-tofu-state"
    key            = "environments/dev/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tofu-state-lock"
  }
}
```

```hcl
# environments/prod/backend.tf
terraform {
  backend "s3" {
    bucket         = "my-company-tofu-state"
    key            = "environments/prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tofu-state-lock"
  }
}
```

## Partial Backend Configuration with -backend-config

Use partial backend configuration for flexibility:

```hcl
# backend.tf (shared, checked into VCS)
terraform {
  backend "s3" {
    bucket         = "my-company-tofu-state"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tofu-state-lock"
    # key is provided at init time
  }
}
```

```bash
# Initialize each environment with its specific state key
tofu -chdir=environments/dev init \
  -backend-config="key=environments/dev/terraform.tfstate"

tofu -chdir=environments/prod init \
  -backend-config="key=environments/prod/terraform.tfstate"
```

## IAM Permissions Per Environment

Use separate IAM roles per environment to enforce access control:

```hcl
# environments/prod/provider.tf
provider "aws" {
  region = "us-east-1"

  # Assume the production deployment role
  assume_role {
    role_arn     = "arn:aws:iam::PROD_ACCOUNT_ID:role/opentofu-deployer"
    session_name = "opentofu-prod-deploy"
  }
}
```

## State Bucket Setup

Create the state infrastructure with a separate bootstrap configuration:

```hcl
# bootstrap/main.tf
resource "aws_s3_bucket" "state" {
  bucket = "my-company-tofu-state"

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.state.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "AES256" }
  }
}

resource "aws_dynamodb_table" "lock" {
  name         = "tofu-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## Conclusion

Environment state separation is the most fundamental state management decision. Use separate S3 keys (and ideally separate AWS accounts) per environment. The directory-based approach makes it immediately clear which environment you are operating on and prevents accidental cross-environment modifications.
