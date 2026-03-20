# How to Fix Authentication Failures During tofu apply

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Troubleshooting, Authentication, Error, AWS, Infrastructure as Code

Description: Learn how to diagnose and fix authentication failures during tofu apply, including expired credentials, missing permissions, and provider configuration issues.

## Introduction

Authentication failures during `tofu apply` are among the most common errors. They manifest as `InvalidClientTokenId`, `AccessDenied`, `UnauthorizedOperation`, or `403 Forbidden` errors and can be caused by expired credentials, wrong profiles, or missing IAM permissions.

## Diagnosing Authentication Errors

Enable debug logging to see the full error context:

```bash
TF_LOG=DEBUG tofu apply 2>&1 | grep -iE "auth|credential|token|403|401|access denied"
```

## Fix 1: Expired or Missing Credentials (AWS)

```bash
# Check current credentials

aws sts get-caller-identity

# If expired, refresh them
# For SSO:
aws sso login --profile my-profile
export AWS_PROFILE=my-profile

# For static credentials:
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

# Re-run
tofu apply
```

## Fix 2: Wrong AWS Profile

```hcl
# Specify the profile in the provider block
provider "aws" {
  region  = "us-east-1"
  profile = "production"  # Must exist in ~/.aws/config
}
```

Or override at the command line:

```bash
AWS_PROFILE=production tofu apply
```

## Fix 3: IAM Permission Denied

An `AccessDenied` error means the identity is authenticated but lacks the required IAM permission:

```bash
# Find what action was denied
TF_LOG=DEBUG tofu apply 2>&1 | grep "AccessDenied\|is not authorized"
# Look for: User: arn:aws:iam::... is not authorized to perform: ec2:CreateVpc
```

Add the missing permission to the IAM role:

```hcl
resource "aws_iam_role_policy" "tofu_vpc" {
  name = "tofu-vpc-permissions"
  role = aws_iam_role.tofu.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["ec2:CreateVpc", "ec2:DescribeVpcs", "ec2:DeleteVpc"]
      Resource = "*"
    }]
  })
}
```

## Fix 4: GCP Authentication

```bash
# Authenticate with GCP
gcloud auth application-default login

# Or for service accounts:
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"

# Verify
gcloud auth application-default print-access-token
```

## Fix 5: Azure Authentication

```bash
# Login with Azure CLI
az login

# Or use a service principal:
export ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
export ARM_CLIENT_SECRET="your-secret"
export ARM_TENANT_ID="00000000-0000-0000-0000-000000000000"
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
```

## Fix 6: OIDC Token Expiry in CI/CD

OIDC tokens used for GitHub Actions are short-lived. If your apply takes longer than the token lifetime:

```yaml
# .github/workflows/infra.yml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/github-actions-role
    aws-region: us-east-1
    # Role session duration - increase if apply takes a long time
    role-duration-seconds: 3600
```

## Conclusion

Authentication failures break down into: expired credentials (refresh them), wrong profile (check provider configuration), missing IAM permissions (examine the error for the denied action), or CI/CD token expiry (increase session duration). Enable `TF_LOG=DEBUG` to expose the exact API error message that reveals which case you are facing.
