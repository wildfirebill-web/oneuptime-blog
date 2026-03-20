# How to Deploy Velero on Kubernetes with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, Velero, Backup, Disaster Recovery, OpenTofu, Helm, S3

Description: Learn how to deploy Velero on Kubernetes using OpenTofu and Helm for cluster backup and restore, with scheduled backups to S3 and cross-cluster migration support.

## Overview

Velero backs up Kubernetes cluster resources and persistent volumes to object storage. It supports scheduled backups, namespace migration, and disaster recovery. OpenTofu deploys Velero via Helm with AWS S3 as the backup storage location using IRSA for secure access.

## Step 1: Create S3 Bucket and IAM Role

```hcl
# main.tf - S3 bucket for Velero backups
resource "aws_s3_bucket" "velero" {
  bucket = "my-cluster-velero-backups"
}

resource "aws_s3_bucket_versioning" "velero" {
  bucket = aws_s3_bucket.velero.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "velero" {
  bucket = aws_s3_bucket.velero.id

  rule {
    id     = "delete-old-backups"
    status = "Enabled"

    expiration {
      days = 90  # Keep backups for 90 days
    }
  }
}

# IAM role for Velero (IRSA)
resource "aws_iam_role" "velero" {
  name = "velero"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.eks.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${local.oidc_provider}:sub" = "system:serviceaccount:velero:velero"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "velero" {
  role = aws_iam_role.velero.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:DescribeVolumes",
          "ec2:DescribeSnapshots",
          "ec2:CreateTags",
          "ec2:CreateVolume",
          "ec2:CreateSnapshot",
          "ec2:DeleteSnapshot"
        ]
        Resource = ["*"]
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject", "s3:DeleteObject", "s3:PutObject",
          "s3:AbortMultipartUpload", "s3:ListMultipartUploadParts"
        ]
        Resource = ["${aws_s3_bucket.velero.arn}/*"]
      },
      {
        Effect   = "Allow"
        Action   = ["s3:ListBucket"]
        Resource = [aws_s3_bucket.velero.arn]
      }
    ]
  })
}
```

## Step 2: Deploy Velero with Helm

```hcl
resource "helm_release" "velero" {
  name             = "velero"
  repository       = "https://vmware-tanzu.github.io/helm-charts"
  chart            = "velero"
  version          = "6.0.0"
  namespace        = "velero"
  create_namespace = true

  values = [yamlencode({
    initContainers = [{
      name  = "velero-plugin-for-aws"
      image = "velero/velero-plugin-for-aws:v1.9.0"
      volumeMounts = [{
        mountPath = "/target"
        name      = "plugins"
      }]
    }]

    # AWS configuration
    configuration = {
      backupStorageLocation = [{
        name     = "default"
        provider = "aws"
        bucket   = aws_s3_bucket.velero.id
        config = {
          region = "us-east-1"
        }
      }]

      volumeSnapshotLocation = [{
        name     = "default"
        provider = "aws"
        config = {
          region = "us-east-1"
        }
      }]
    }

    serviceAccount = {
      server = {
        annotations = {
          "eks.amazonaws.com/role-arn" = aws_iam_role.velero.arn
        }
      }
    }

    resources = {
      requests = { cpu = "500m", memory = "128Mi" }
      limits   = { cpu = "1000m", memory = "512Mi" }
    }

    metrics = {
      enabled = true
      serviceMonitor = {
        enabled = true
      }
    }
  })]
}
```

## Step 3: Create Scheduled Backups

```hcl
# Daily full cluster backup
resource "kubernetes_manifest" "daily_backup_schedule" {
  depends_on = [helm_release.velero]

  manifest = {
    apiVersion = "velero.io/v1"
    kind       = "Schedule"
    metadata = {
      name      = "daily-full-backup"
      namespace = "velero"
    }
    spec = {
      schedule = "0 2 * * *"  # 2 AM daily

      template = {
        ttl                     = "720h"  # 30 days retention
        includedNamespaces      = ["*"]
        excludedNamespaces      = ["kube-system", "velero"]
        storageLocation         = "default"
        volumeSnapshotLocations = ["default"]

        # Labels to include in backup
        labelSelector = null
      }

      useOwnerReferencesInBackup = false
    }
  }
}

# Hourly backup for critical namespaces
resource "kubernetes_manifest" "hourly_critical_backup" {
  manifest = {
    apiVersion = "velero.io/v1"
    kind       = "Schedule"
    metadata = {
      name      = "hourly-critical"
      namespace = "velero"
    }
    spec = {
      schedule = "0 * * * *"  # Every hour

      template = {
        ttl                = "168h"  # 7 days retention
        includedNamespaces = ["production", "database"]
      }
    }
  }
}
```

## Summary

Velero deployed with OpenTofu provides Kubernetes cluster backup and disaster recovery backed by S3. Scheduled backups protect both cluster resources and persistent volume snapshots. The namespace-based backup model enables both full cluster recovery and surgical restoration of specific workloads, making Velero ideal for cluster migrations, DR testing, and protecting stateful workloads.
