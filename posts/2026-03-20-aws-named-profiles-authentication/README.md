# How to Authenticate with AWS Using Named Profiles

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, CLI, Named Profiles, Authentication, Configuration, DevOps

Description: Learn how to configure and use AWS named profiles to manage multiple AWS accounts and roles from a single workstation.

---

AWS named profiles let you store multiple sets of credentials and configuration in `~/.aws/credentials` and `~/.aws/config`, then switch between them using the `--profile` flag or `AWS_PROFILE` environment variable.

---

## Configure Named Profiles

```bash
# Interactive setup

aws configure --profile production
aws configure --profile staging
aws configure --profile development
```

Or edit the files directly:

```ini
# ~/.aws/credentials
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCY

[production]
aws_access_key_id = AKIAPROD1234567890AB
aws_secret_access_key = prodSecretKeyHere

[staging]
aws_access_key_id = AKIASTAGING123456789
aws_secret_access_key = stagingSecretKeyHere
```

```ini
# ~/.aws/config
[default]
region = us-east-1
output = json

[profile production]
region = us-east-1
output = json

[profile staging]
region = eu-west-1
output = json
```

---

## Use a Named Profile

```bash
# With the --profile flag
aws s3 ls --profile production
aws ec2 describe-instances --profile staging

# Set as environment variable for the session
export AWS_PROFILE=production
aws sts get-caller-identity
```

---

## Role Assumption with Named Profiles

```ini
# ~/.aws/config
[profile prod-admin]
role_arn = arn:aws:iam::123456789012:role/AdminRole
source_profile = default
mfa_serial = arn:aws:iam::987654321098:mfa/myuser
```

```bash
aws sts get-caller-identity --profile prod-admin
# Prompts for MFA token, then assumes the role
```

---

## Use Named Profiles with OpenTofu

```hcl
provider "aws" {
  profile = "production"
  region  = "us-east-1"
}
```

Or via environment:
```bash
export AWS_PROFILE=production
tofu apply
```

---

## Summary

AWS named profiles separate credentials and regional settings per account or role in `~/.aws/credentials` and `~/.aws/config`. Switch profiles with `--profile` or `AWS_PROFILE`. Use profiles with role assumption and MFA to enable secure cross-account workflows from a single workstation.
