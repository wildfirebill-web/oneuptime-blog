# How to Set Up EKS Secrets Encryption with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, EKS, Secrets Encryption, KMS, Kubernetes, Security, Infrastructure as Code

Description: Learn how to enable envelope encryption for Kubernetes Secrets in EKS using AWS KMS keys with OpenTofu to protect sensitive data stored in etcd.

## Introduction

By default, Kubernetes Secrets are stored in base64-encoded format in etcd-not encrypted. EKS envelope encryption uses AWS KMS to encrypt the envelope encryption key that protects Secrets, providing a strong security layer for credentials, tokens, and certificates stored as Kubernetes Secrets.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with EKS and KMS permissions
- Note: Encryption can be enabled at cluster creation or enabled later but cannot be disabled once enabled

## Step 1: Create a KMS Key for Secrets Encryption

```hcl
# Dedicated KMS key for encrypting Kubernetes Secrets

resource "aws_kms_key" "eks_secrets" {
  description             = "KMS key for EKS secrets encryption - ${var.cluster_name}"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  # Key policy must allow the EKS service and authorized users
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow EKS to use this key"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.eks_cluster.arn
        }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      }
    ]
  })

  tags = {
    Name    = "${var.cluster_name}-secrets-encryption"
    Cluster = var.cluster_name
    Purpose = "SecretsEncryption"
  }
}

resource "aws_kms_alias" "eks_secrets" {
  name          = "alias/${var.cluster_name}-secrets"
  target_key_id = aws_kms_key.eks_secrets.key_id
}
```

## Step 2: Create EKS Cluster with Secrets Encryption

```hcl
resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids              = var.private_subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = false
  }

  # Enable envelope encryption for Kubernetes Secrets
  encryption_config {
    # Encrypt the "secrets" resource type
    resources = ["secrets"]

    provider {
      key_arn = aws_kms_key.eks_secrets.arn
    }
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator"]

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
    aws_kms_key.eks_secrets
  ]

  tags = {
    Name               = var.cluster_name
    SecretsEncryption  = "enabled"
    KMSKeyId           = aws_kms_key.eks_secrets.key_id
  }
}
```

## Step 3: Enable Encryption on an Existing Cluster

```hcl
# Use the AWS CLI to enable encryption on an existing cluster
# This cannot be done via the aws_eks_cluster resource after creation
# but can be done via AWS CLI or SDK

resource "null_resource" "enable_encryption" {
  triggers = {
    cluster_name = var.cluster_name
    kms_key_arn  = aws_kms_key.eks_secrets.arn
  }

  provisioner "local-exec" {
    command = <<-EOF
      aws eks associate-encryption-config \
        --cluster-name ${var.cluster_name} \
        --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"${aws_kms_key.eks_secrets.arn}"}}]' \
        --region ${var.region}
    EOF
  }
}
```

## Step 4: Verify Encryption is Working

```bash
# Create a test secret
kubectl create secret generic test-secret \
  --from-literal=password=mysecretpassword

# Verify it's encrypted (should show kms provider in etcd)
kubectl describe secret test-secret

# Check cluster encryption config
aws eks describe-cluster \
  --name my-cluster \
  --query 'cluster.encryptionConfig'
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

EKS envelope encryption protects Kubernetes Secrets using AWS KMS, ensuring that even if etcd storage were compromised, secrets would remain encrypted. Once enabled, encryption cannot be disabled, so plan your KMS key management carefully-including key rotation policies, backup, and cross-account access if needed. Monitor KMS API call volumes in CloudTrail as each secret creation or retrieval generates KMS API calls.
