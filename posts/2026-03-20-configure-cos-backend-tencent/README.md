# How to Configure the COS Backend (Tencent Cloud) in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Tencent Cloud, State Management

Description: Learn how to configure the OpenTofu COS backend to store state files in Tencent Cloud Object Storage with versioning, encryption, and state locking.

## Introduction

The COS (Cloud Object Storage) backend stores OpenTofu state in Tencent Cloud Object Storage. It's the recommended remote state backend for infrastructure deployed on Tencent Cloud, providing versioning, encryption, and server-side locking support.

## Prerequisites

- Tencent Cloud account with COS service enabled
- SecretId and SecretKey or CAM role with COS permissions
- COS bucket created in your target region

## Step 1: Create the COS Bucket

```bash
# Using tccli (Tencent Cloud CLI)
tccli cos create-bucket \
  --bucket my-terraform-state-1234567890 \
  --region ap-guangzhou

# Enable versioning
tccli cos put-bucket-versioning \
  --bucket my-terraform-state-1234567890 \
  --region ap-guangzhou \
  --status Enabled
```

Or via the Tencent Cloud Console: COS → Bucket List → Create Bucket

## Step 2: Configure the COS Backend

```hcl
# backend.tf
terraform {
  backend "cos" {
    region = "ap-guangzhou"         # COS region
    bucket = "my-terraform-state-1234567890"  # Bucket name (includes AppID)
    prefix = "terraform/state/prod" # Path prefix in the bucket

    # Optional: enable server-side encryption
    encrypt = true

    # Optional: use custom endpoint
    # endpoint = "cos.ap-guangzhou.myqcloud.com"
  }
}
```

## Authentication Configuration

### Using Environment Variables

```bash
# Set credentials via environment variables
export TENCENTCLOUD_SECRET_ID="AKID..."
export TENCENTCLOUD_SECRET_KEY="your-secret-key"
export TENCENTCLOUD_REGION="ap-guangzhou"
```

### Using Credentials in Backend Config

```hcl
terraform {
  backend "cos" {
    region     = "ap-guangzhou"
    bucket     = "my-terraform-state-1234567890"
    prefix     = "terraform/state/prod"
    secret_id  = var.tencent_secret_id   # Use var or env var, not hardcoded
    secret_key = var.tencent_secret_key
  }
}
```

### Using CAM Role (Recommended for CVM)

When running on a Tencent Cloud CVM with a CAM role attached:

```bash
# No credentials needed — automatically uses the CVM's CAM role
export TENCENTCLOUD_REGION="ap-guangzhou"
```

## State File Organization

```
COS bucket: my-terraform-state-1234567890
├── terraform/state/prod/terraform.tfstate
├── terraform/state/staging/terraform.tfstate
└── terraform/state/networking/terraform.tfstate
```

## Setting Up Permissions

Create a CAM policy with minimal COS permissions:

```json
{
  "version": "2.0",
  "statement": [
    {
      "effect": "allow",
      "action": [
        "cos:PutObject",
        "cos:GetObject",
        "cos:DeleteObject",
        "cos:ListObjects"
      ],
      "resource": [
        "qcs::cos:ap-guangzhou:uid/1234567890:my-terraform-state-1234567890/*"
      ]
    }
  ]
}
```

## Server-Side Encryption

```hcl
terraform {
  backend "cos" {
    region  = "ap-guangzhou"
    bucket  = "my-terraform-state-1234567890"
    prefix  = "terraform/state/prod"
    encrypt = true  # SSE-COS (Tencent managed keys)
  }
}
```

For Customer-Managed Keys (SSE-KMS):

```hcl
terraform {
  backend "cos" {
    region         = "ap-guangzhou"
    bucket         = "my-terraform-state-1234567890"
    prefix         = "terraform/state/prod"
    encrypt        = true
    kms_key_id     = "your-kms-key-id"  # KMS key for SSE-KMS
  }
}
```

## Workspace Configuration

```bash
# Create workspaces
tofu workspace new production
tofu workspace new staging

# State file paths:
# terraform/state/prod/terraform.tfstate       ← default
# terraform/state/prod/env:/production/terraform.tfstate  ← production workspace
```

## Initialize and Verify

```bash
# Set credentials
export TENCENTCLOUD_SECRET_ID="AKID..."
export TENCENTCLOUD_SECRET_KEY="your-secret-key"

# Initialize
tofu init

# Verify state is accessible
tofu state list

# Check bucket contents
tccli cos list-objects \
  --bucket my-terraform-state-1234567890 \
  --region ap-guangzhou \
  --prefix terraform/state/
```

## Conclusion

The COS backend provides a native Tencent Cloud state storage solution that integrates with the Tencent Cloud IAM and encryption services. Enable versioning for state history, use server-side encryption for data protection, and apply CAM policies following least-privilege principles. For compute instances on CVM, use CAM roles instead of static credentials for improved security.
