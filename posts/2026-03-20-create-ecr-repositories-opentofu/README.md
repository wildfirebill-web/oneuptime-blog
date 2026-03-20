# How to Create AWS ECR Repositories with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, ECR, Container Registry, Infrastructure as Code

Description: Learn how to create AWS Elastic Container Registry repositories with OpenTofu for storing, versioning, and deploying container images.

AWS ECR is a managed container registry that integrates with ECS, EKS, and Lambda. Managing repositories in OpenTofu ensures consistent scanning, encryption, and access policies across all your container images.

## Creating an ECR Repository

```hcl
resource "aws_ecr_repository" "api" {
  name                 = "myapp/api"
  image_tag_mutability = "IMMUTABLE"  # Prevent overwriting tags

  image_scanning_configuration {
    scan_on_push = true  # Scan every pushed image for vulnerabilities
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Service     = "api"
    Environment = "production"
  }
}
```

## Repository Policy

```hcl
resource "aws_ecr_repository_policy" "api" {
  repository = aws_ecr_repository.api.name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Allow CI/CD to push images
      {
        Sid    = "AllowCIPush"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.ci_role.arn
        }
        Action = [
          "ecr:BatchCheckLayerAvailability",
          "ecr:CompleteLayerUpload",
          "ecr:InitiateLayerUpload",
          "ecr:PutImage",
          "ecr:UploadLayerPart",
        ]
      },
      # Allow ECS/EKS to pull images
      {
        Sid    = "AllowServicePull"
        Effect = "Allow"
        Principal = {
          AWS = [
            aws_iam_role.ecs_task.arn,
            aws_iam_role.eks_nodes.arn,
          ]
        }
        Action = [
          "ecr:BatchGetImage",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchCheckLayerAvailability",
        ]
      },
      # Allow cross-account pull from staging account
      {
        Sid    = "AllowCrossAccountPull"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.staging_account_id}:root"
        }
        Action = [
          "ecr:BatchGetImage",
          "ecr:GetDownloadUrlForLayer",
        ]
      }
    ]
  })
}
```

## Multiple Repositories

```hcl
locals {
  services = ["api", "worker", "scheduler", "frontend", "admin"]
}

resource "aws_ecr_repository" "services" {
  for_each = toset(local.services)

  name                 = "myapp/${each.key}"
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }

  tags = {
    Service = each.key
  }
}
```

## Outputs

```hcl
output "repository_urls" {
  description = "ECR repository URLs for each service"
  value = {
    for k, v in aws_ecr_repository.services : k => v.repository_url
  }
}

output "registry_id" {
  value = aws_ecr_repository.api.registry_id
}
```

## Conclusion

AWS ECR repositories in OpenTofu provide secure, managed container storage. Always use IMMUTABLE image tags to prevent accidental overwrites, enable scan-on-push for automatic vulnerability detection, and configure repository policies to grant least-privilege access - push for CI/CD, pull for compute roles. Use for_each to manage all service repositories with consistent settings.
