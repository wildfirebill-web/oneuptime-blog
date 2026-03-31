# How to Use Dapr AWS Bindings with Instance Profiles

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, AWS, EC2, Instance Profile, Binding

Description: Learn how to configure Dapr AWS bindings to use EC2 instance profiles and ECS task roles for automatic, credential-free AWS authentication in cloud deployments.

---

## What Are Instance Profiles?

An EC2 instance profile is a container for an IAM role that allows applications running on an EC2 instance to obtain temporary AWS credentials from the Instance Metadata Service (IMDS) automatically. The application never needs to manage credentials directly - they are rotated automatically by AWS.

Dapr AWS bindings support instance profiles out of the box by leaving the `accessKey` and `secretKey` fields empty (or omitting them), causing the AWS SDK to fall through to IMDS.

## Configuring Dapr Bindings for Instance Profile Auth

Simply omit the credential fields from your binding component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: order-store
  namespace: default
spec:
  type: bindings.aws.dynamodb
  version: v1
  metadata:
    - name: table
      value: "orders"
    - name: region
      value: "us-east-1"
```

When running on an EC2 instance with an attached IAM role, Dapr fetches credentials automatically from IMDS.

## Setting Up the EC2 Instance Profile

### Step 1: Create the IAM Role

```bash
aws iam create-role \
  --role-name dapr-ec2-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'
```

### Step 2: Attach Policies

```bash
# DynamoDB access
aws iam put-role-policy \
  --role-name dapr-ec2-role \
  --policy-name dynamodb-orders \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "dynamodb:PutItem",
          "dynamodb:GetItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:Query",
          "dynamodb:Scan"
        ],
        "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/orders"
      }
    ]
  }'

# S3 access
aws iam put-role-policy \
  --role-name dapr-ec2-role \
  --policy-name s3-reports \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["s3:PutObject", "s3:GetObject"],
        "Resource": "arn:aws:s3:::my-reports/*"
      }
    ]
  }'
```

### Step 3: Create the Instance Profile

```bash
aws iam create-instance-profile \
  --instance-profile-name dapr-ec2-profile

aws iam add-role-to-instance-profile \
  --instance-profile-name dapr-ec2-profile \
  --role-name dapr-ec2-role
```

### Step 4: Attach to EC2 Instance

```bash
aws ec2 associate-iam-instance-profile \
  --instance-id i-1234567890abcdef0 \
  --iam-instance-profile Name=dapr-ec2-profile
```

## Using Task Roles for ECS

For ECS, use a task role instead of instance profiles - this gives per-task credential isolation:

```bash
aws iam create-role \
  --role-name dapr-ecs-task-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "ecs-tasks.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'
```

Reference the task role in your ECS task definition:

```json
{
  "family": "dapr-order-service",
  "taskRoleArn": "arn:aws:iam::123456789012:role/dapr-ecs-task-role",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "order-service",
      "image": "myrepo/order-service:latest",
      "environment": [
        {"name": "APP_PORT", "value": "3000"}
      ]
    },
    {
      "name": "daprd",
      "image": "daprio/daprd:latest",
      "command": ["./daprd", "-app-id", "order-service", "-app-port", "3000"]
    }
  ]
}
```

## IMDSv2 Compatibility

AWS recommends IMDSv2 (token-based IMDS). Dapr's AWS SDK automatically supports IMDSv2. Enforce it on your instances:

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-1234567890abcdef0 \
  --http-tokens required \
  --http-put-response-hop-limit 1
```

## Verifying Credential Resolution

Test that credentials resolve correctly from within the EC2 instance:

```bash
# Verify the identity
aws sts get-caller-identity --region us-east-1

# Test access to a specific resource
aws dynamodb list-tables --region us-east-1
```

## Summary

EC2 instance profiles and ECS task roles eliminate static credential management for Dapr AWS bindings. Simply omit `accessKey` and `secretKey` from your component spec, attach the appropriate IAM role to your EC2 instance or ECS task, and Dapr automatically obtains temporary credentials from IMDS or the ECS credentials endpoint. This approach is simpler, more secure, and requires no secret rotation infrastructure.
