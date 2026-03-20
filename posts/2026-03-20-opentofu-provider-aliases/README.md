---
title: "Using Provider Aliases in OpenTofu"
author: nawazdhandala
tags: opentofu, terraform, iac, providers, aliases
description: "Learn how to use provider aliases in OpenTofu to manage multiple configurations of the same provider within a single workspace."
---

# Using Provider Aliases in OpenTofu

Provider aliases let you use multiple configurations of the same provider in a single OpenTofu workspace. This is essential for multi-region deployments, cross-account setups, and managing resources in different environments simultaneously.

## Creating a Provider Alias

```hcl
# Default provider (no alias)
provider "aws" {
  region = "us-east-1"
}

# Aliased provider — explicitly named
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}
```

## Using Aliases in Resources

```hcl
# Uses the default provider (us-east-1)
resource "aws_instance" "east" {
  ami           = "ami-east-12345"
  instance_type = "t3.medium"

  tags = {
    Region = "us-east-1"
  }
}

# Explicitly references the aliased provider (us-west-2)
resource "aws_instance" "west" {
  provider      = aws.west
  ami           = "ami-west-12345"
  instance_type = "t3.medium"

  tags = {
    Region = "us-west-2"
  }
}
```

## Multi-Region S3 Replication

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "dr"
  region = "eu-west-1"
}

resource "aws_s3_bucket" "primary" {
  bucket = "myapp-primary-data"
}

resource "aws_s3_bucket" "replica" {
  provider = aws.dr
  bucket   = "myapp-replica-data"
}

resource "aws_s3_bucket_replication_configuration" "main" {
  bucket = aws_s3_bucket.primary.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.replica.arn
      storage_class = "STANDARD_IA"
    }
  }
}
```

## Cross-Account Deployments

```hcl
provider "aws" {
  region = "us-east-1"
  # Uses default credentials (dev account)
}

provider "aws" {
  alias  = "prod"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::PROD_ACCOUNT_ID:role/TerraformDeployRole"
    session_name = "terraform-deployment"
  }
}

# Create in dev account
resource "aws_s3_bucket" "dev_artifacts" {
  bucket = "dev-build-artifacts"
}

# Create in prod account
resource "aws_s3_bucket" "prod_artifacts" {
  provider = aws.prod
  bucket   = "prod-build-artifacts"
}
```

## Passing Aliases to Modules

When a module needs a non-default provider, you must explicitly pass it:

```hcl
# Root configuration
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "replica"
  region = "eu-central-1"
}

module "database" {
  source = "./modules/rds"

  providers = {
    aws         = aws            # Default provider
    aws.replica = aws.replica    # Aliased provider
  }

  primary_region = "us-east-1"
  replica_region = "eu-central-1"
}
```

```hcl
# modules/rds/versions.tf — declare expected providers
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = ">= 4.0"
      configuration_aliases = [aws.replica]  # Declares that this module uses aws.replica
    }
  }
}

# modules/rds/main.tf
resource "aws_db_instance" "primary" {
  # Uses default aws provider
  engine         = "postgres"
  instance_class = "db.r5.large"
}

resource "aws_db_instance" "replica" {
  provider            = aws.replica  # Uses the passed alias
  replicate_source_db = aws_db_instance.primary.arn
  instance_class      = "db.r5.large"
}
```

## Multiple Kubernetes Clusters

```hcl
provider "kubernetes" {
  alias                  = "cluster_a"
  host                   = data.aws_eks_cluster.cluster_a.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster_a.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.cluster_a.token
}

provider "kubernetes" {
  alias                  = "cluster_b"
  host                   = data.aws_eks_cluster.cluster_b.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster_b.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.cluster_b.token
}

resource "kubernetes_namespace" "monitoring_a" {
  provider = kubernetes.cluster_a

  metadata {
    name = "monitoring"
  }
}

resource "kubernetes_namespace" "monitoring_b" {
  provider = kubernetes.cluster_b

  metadata {
    name = "monitoring"
  }
}
```

## Conclusion

Provider aliases are powerful for multi-region and cross-account architectures. Use them to manage resources across different configurations of the same provider while keeping all your infrastructure in a single workspace. Remember to declare `configuration_aliases` in modules that accept non-default provider configurations.
