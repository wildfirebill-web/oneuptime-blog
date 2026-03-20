# How to Add AWS ECR as a Registry in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, AWS, ECR, Registry, DevOps

Description: Learn how to configure Amazon Elastic Container Registry (ECR) as a registry in Portainer for pulling private images from AWS.

## Introduction

Amazon Elastic Container Registry (ECR) is a fully managed Docker registry service. ECR uses IAM-based authentication with temporary tokens that expire every 12 hours, making it different from username/password registries. Portainer supports ECR with automatic token refresh. This guide covers configuring AWS ECR in Portainer.

## Prerequisites

- Portainer CE or BE installed
- An AWS account with ECR repositories
- AWS Access Key ID and Secret Access Key (or IAM role if Portainer runs on EC2)
- Admin access to Portainer

## Understanding ECR Authentication

ECR tokens are temporary (12 hours) and require AWS credentials to generate. Portainer BE handles token refresh automatically. For CE, you may need to manually refresh tokens or use a workaround.

## Step 1: Create an IAM User for Portainer

Create a dedicated IAM user with minimal ECR permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:DescribeImages",
        "ecr:DescribeRepositories",
        "ecr:ListImages"
      ],
      "Resource": "*"
    }
  ]
}
```

Save this as a policy named `PortainerECRReadOnly` and attach it to your IAM user.

```bash
# Using AWS CLI to create the policy

aws iam create-policy \
  --policy-name PortainerECRReadOnly \
  --policy-document file://ecr-policy.json

# Create IAM user
aws iam create-user --user-name portainer-ecr

# Attach policy
aws iam attach-user-policy \
  --user-name portainer-ecr \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/PortainerECRReadOnly

# Create access key
aws iam create-access-key --user-name portainer-ecr
# Save the AccessKeyId and SecretAccessKey
```

## Step 2: Add ECR Registry in Portainer

1. Go to **Registries** in Portainer
2. Click **+ Add registry**
3. Select **AWS Elastic Container Registry**

## Step 3: Fill in ECR Configuration

```text
Registry type:  AWS ECR
Region:         us-east-1         (your AWS region)
Access key ID:  AKIAIOSFODNN7EXAMPLE
Secret key:     wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

For cross-account ECR access:

```text
Registry URL:   123456789012.dkr.ecr.us-east-1.amazonaws.com
```

4. Click **Add registry**

## Step 4: Get the ECR Registry URL

Your ECR registry URL follows this format:

```text
{account-id}.dkr.ecr.{region}.amazonaws.com
```

Find it in the AWS Console:
1. Go to **Amazon ECR** in the AWS Console
2. Click on a repository
3. Copy the **URI** prefix up to the repository name

```bash
# Or via CLI
aws ecr describe-repositories --region us-east-1 \
  --query 'repositories[0].repositoryUri' --output text
# Output: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myrepo
```

## Step 5: Use ECR Images in Portainer Stacks

```yaml
version: "3.8"

services:
  app:
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
    # Portainer uses stored ECR credentials and handles token refresh

  worker:
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myworker:v2.0
```

## Step 6: Handle ECR Token Expiration (Portainer CE)

ECR tokens expire after 12 hours. In Portainer CE, you need to refresh manually or use a cron job:

```bash
#!/bin/bash
# refresh-ecr-token.sh
# Run every 6 hours via cron

AWS_REGION="us-east-1"
AWS_ACCOUNT="123456789012"

# Get ECR token
TOKEN=$(aws ecr get-login-password --region $AWS_REGION)

# Update Docker credentials
docker login \
  --username AWS \
  --password-stdin \
  ${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com << EOF
$TOKEN
EOF

echo "ECR token refreshed at $(date)"
```

```bash
# Cron job - refresh every 6 hours
0 */6 * * * /opt/scripts/refresh-ecr-token.sh >> /var/log/ecr-refresh.log 2>&1
```

## Step 7: Use IAM Roles (EC2/ECS/EKS)

If Portainer runs on an EC2 instance, use an IAM role instead of access keys:

```bash
# Attach IAM role to EC2 instance with ECR read permissions
# The instance automatically uses the role's credentials
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com
```

In Portainer settings, leave the access key fields empty and Portainer will use the instance metadata.

## Step 8: Multi-Region ECR

If you have repositories in multiple AWS regions, add a separate registry entry for each region:

```text
Registry 1: us-east-1 ECR (production)
  Region: us-east-1
  Access Key: AKIA...

Registry 2: eu-west-1 ECR (Europe)
  Region: eu-west-1
  Access Key: AKIA...
```

## Troubleshooting

### No Basic Auth Credentials

```text
Error: no basic auth credentials
```

ECR token has expired. Refresh it manually or via cron (CE) or wait for Portainer BE auto-refresh.

### Access Denied

```text
Error: AccessDeniedException: User is not authorized to perform: ecr:GetAuthorizationToken
```

Add `ecr:GetAuthorizationToken` to the IAM policy.

### Image Not Found

```bash
Error: repository does not exist or may require 'docker login'
```

Verify the full image URI including account ID and region.

## Conclusion

Integrating AWS ECR with Portainer requires understanding ECR's token-based authentication model. For Portainer BE, this is largely automated. For CE, set up a token refresh cron job to ensure Portainer always has valid credentials. Using IAM users with minimal permissions (read-only ECR access) follows the principle of least privilege for your container infrastructure.
