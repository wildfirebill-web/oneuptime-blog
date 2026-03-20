# How to Authenticate with AWS Using Environment Variables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, Authentication, Environment Variables, CLI, Security, DevOps

Description: Learn how to configure AWS SDK and CLI authentication using environment variables for local development and CI/CD pipelines.

---

Environment variables are the simplest way to supply AWS credentials without modifying configuration files. They work across all AWS SDKs, the AWS CLI, OpenTofu, and any tool that uses the AWS SDKs.

---

## Required Environment Variables

```bash
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_DEFAULT_REGION=us-east-1
```

---

## Optional Session Token (for Temporary Credentials)

```bash
export AWS_SESSION_TOKEN=AQoDYXdzEJr...
```

Temporary credentials from STS or IAM role assumption require `AWS_SESSION_TOKEN`.

---

## Verify Authentication

```bash
aws sts get-caller-identity
# {
#   "UserId": "AIDACKCEVSQ6C2EXAMPLE",
#   "Account": "123456789012",
#   "Arn": "arn:aws:iam::123456789012:user/myuser"
# }
```

---

## Use in a Shell Script

```bash
#!/bin/bash
export AWS_ACCESS_KEY_ID="${CI_AWS_ACCESS_KEY_ID}"
export AWS_SECRET_ACCESS_KEY="${CI_AWS_SECRET_ACCESS_KEY}"
export AWS_DEFAULT_REGION="us-east-1"

aws s3 ls s3://my-bucket
```

---

## Use in Docker

```dockerfile
ENV AWS_ACCESS_KEY_ID=""
ENV AWS_SECRET_ACCESS_KEY=""
ENV AWS_DEFAULT_REGION="us-east-1"
```

Or pass at runtime:

```bash
docker run -e AWS_ACCESS_KEY_ID=... -e AWS_SECRET_ACCESS_KEY=... myimage
```

---

## Credential Precedence

AWS uses this lookup order:
1. Environment variables
2. `~/.aws/credentials` file
3. IAM instance profile / ECS task role
4. IAM role for service accounts (EKS)

Environment variables take highest priority (after explicit code overrides), making them ideal for overriding config file credentials during testing.

---

## Summary

Set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_DEFAULT_REGION` to authenticate any AWS SDK or CLI tool. Add `AWS_SESSION_TOKEN` for temporary credentials. Environment variable credentials take precedence over config file credentials, making them well-suited for CI/CD pipelines where secrets are injected as environment variables.
