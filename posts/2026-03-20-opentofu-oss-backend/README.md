# How to Configure the OSS Backend (Alibaba Cloud) in OpenTofu - Opentofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Backend

Description: Learn how to configure the OSS backend in OpenTofu to store state in Alibaba Cloud Object Storage Service with built-in locking via TableStore.

## Introduction

The OSS backend stores OpenTofu state in Alibaba Cloud Object Storage Service (OSS). It optionally uses Alibaba Cloud TableStore for state locking. This is the standard remote backend for teams building on Alibaba Cloud infrastructure.

## Basic Configuration

```hcl
terraform {
  backend "oss" {
    region = "cn-hangzhou"
    bucket = "acme-tofu-state"
    prefix = "production"
  }
}
```

## Authentication

```hcl
terraform {
  backend "oss" {
    region            = "cn-hangzhou"
    bucket            = "acme-tofu-state"
    prefix            = "production"
    access_key        = var.alicloud_access_key
    secret_key        = var.alicloud_secret_key
  }
}
```

Or via environment variables:

```bash
export ALICLOUD_ACCESS_KEY="your-access-key"
export ALICLOUD_SECRET_KEY="your-secret-key"
export ALICLOUD_REGION="cn-hangzhou"

tofu init
```

## Creating the OSS Bucket

```bash
# Using aliyun CLI

aliyun oss mb oss://acme-tofu-state --region cn-hangzhou

# Enable versioning
aliyun oss bucket-versioning --method put oss://acme-tofu-state \
  --payer BucketOwner \
  --status Enabled
```

## State Locking with TableStore

OSS uses Alibaba Cloud TableStore for distributed locking:

```hcl
terraform {
  backend "oss" {
    region              = "cn-hangzhou"
    bucket              = "acme-tofu-state"
    prefix              = "production"

    # TableStore for state locking
    tablestore_endpoint = "https://tofu-lock.cn-hangzhou.ots.aliyuncs.com"
    tablestore_table    = "terraform_lock"
  }
}
```

Create the TableStore table:

```bash
# Create TableStore instance and table via console or aliyun CLI
# Table schema: primary key = "LockID" (string)
aliyun tablestore CreateTable \
  --InstanceName tofu-lock \
  --TableMeta.TableName terraform_lock \
  --TableMeta.PrimaryKey.0.Name LockID \
  --TableMeta.PrimaryKey.0.Type STRING
```

## Encryption at Rest

```hcl
terraform {
  backend "oss" {
    region          = "cn-hangzhou"
    bucket          = "acme-tofu-state"
    prefix          = "production"
    encrypt         = true
    acl             = "private"
    server_side_encryption = "AES256"
  }
}
```

## STS Token Authentication (for short-lived credentials)

```bash
# Obtain STS credentials
CREDS=$(aliyun sts AssumeRole --RoleArn acs:ram::123456789:role/TofuStateRole --RoleSessionName tofu-session)

export ALICLOUD_ACCESS_KEY=$(echo $CREDS | jq -r '.Credentials.AccessKeyId')
export ALICLOUD_SECRET_KEY=$(echo $CREDS | jq -r '.Credentials.AccessKeySecret')
export ALICLOUD_SECURITY_TOKEN=$(echo $CREDS | jq -r '.Credentials.SecurityToken')

tofu init
```

## RAM Policy for State Access

```json
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "oss:GetObject",
        "oss:PutObject",
        "oss:DeleteObject",
        "oss:ListObjects"
      ],
      "Resource": [
        "acs:oss:*:*:acme-tofu-state/production/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ots:GetRow",
        "ots:PutRow",
        "ots:UpdateRow",
        "ots:DeleteRow"
      ],
      "Resource": "acs:ots:cn-hangzhou:*:instance/tofu-lock/table/terraform_lock"
    }
  ]
}
```

## Multiple Environments

```hcl
# production/backend.tf
terraform {
  backend "oss" {
    region = "cn-hangzhou"
    bucket = "acme-tofu-state"
    prefix = "production"
    tablestore_endpoint = "https://tofu-lock.cn-hangzhou.ots.aliyuncs.com"
    tablestore_table    = "terraform_lock"
  }
}

# staging/backend.tf
terraform {
  backend "oss" {
    region = "cn-hangzhou"
    bucket = "acme-tofu-state"
    prefix = "staging"
    tablestore_endpoint = "https://tofu-lock.cn-hangzhou.ots.aliyuncs.com"
    tablestore_table    = "terraform_lock"
  }
}
```

## Conclusion

The OSS backend provides scalable state storage for Alibaba Cloud deployments. Enable versioning on the bucket for state history, configure TableStore for distributed locking to prevent concurrent modifications, and use RAM policies to restrict access. Use STS tokens for short-lived credentials in CI/CD pipelines.
