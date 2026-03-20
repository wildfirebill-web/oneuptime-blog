# How to Add AWS ECR as a Registry in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, AWS ECR, Container Registry, AWS, DevOps

Description: Learn how to connect Amazon Elastic Container Registry (ECR) to Portainer so you can pull private images from AWS.

## Overview

AWS Elastic Container Registry (ECR) is a fully managed container registry. Unlike static username/password credentials, ECR uses short-lived tokens (valid 12 hours) that must be refreshed regularly. Portainer supports ECR natively in Business Edition, and you can also use a workaround for Community Edition.

## Getting ECR Credentials

ECR authentication tokens are obtained using the AWS CLI:

```bash
# Get an ECR login token (valid for 12 hours)

aws ecr get-login-password --region us-east-1

# This outputs a token you use as a password with username "AWS"
```

## Adding ECR in Portainer Business Edition

1. Go to **Settings > Registries** and click **Add registry**.
2. Select **Amazon ECR** as the registry type.
3. Enter:
   - **Registry URL**: Your ECR registry URL (e.g., `123456789012.dkr.ecr.us-east-1.amazonaws.com`)
   - **Region**: Your AWS region (e.g., `us-east-1`)
   - **Access key ID**: Your AWS access key ID
   - **Secret access key**: Your AWS secret access key
4. Portainer will handle token refresh automatically.

## Adding ECR in Community Edition (Manual Token Method)

Since ECR tokens expire after 12 hours, you need to refresh them. Use a cron job or helper container:

```bash
# Script to refresh ECR credentials in Portainer
#!/bin/bash

ECR_TOKEN=$(aws ecr get-login-password --region us-east-1)
PORTAINER_URL="http://localhost:9000"
PORTAINER_TOKEN="your-portainer-api-token"
REGISTRY_ID="1"  # The registry ID in Portainer

# Update the registry credentials via Portainer API
curl -X PUT "${PORTAINER_URL}/api/registries/${REGISTRY_ID}" \
  -H "Authorization: Bearer ${PORTAINER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"Username\": \"AWS\",
    \"Password\": \"${ECR_TOKEN}\"
  }"
```

## Setting Up an IAM Policy for ECR Access

Ensure your AWS credentials have the necessary ECR permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchCheckLayerAvailability"
      ],
      "Resource": "*"
    }
  ]
}
```

## Testing ECR Access

```bash
# Authenticate Docker CLI with ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Pull an image from ECR to verify access
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

## Conclusion

Portainer Business Edition handles ECR token refresh automatically, making it the recommended option for production AWS workloads. For Community Edition, automate token refresh using a scheduled script or the `amazon-ecr-credential-helper`.
