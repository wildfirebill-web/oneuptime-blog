---
title: "Using OCI Registries as Module Sources in OpenTofu"
author: nawazdhandala
tags: opentofu, terraform, iac, modules, oci
description: "Learn how to use OCI (Open Container Initiative) registries as module sources in OpenTofu."
---

# Using OCI Registries as Module Sources in OpenTofu

OpenTofu can source modules from OCI (Open Container Initiative) registries — the same infrastructure used for container images. This allows organizations to use existing registry infrastructure like ECR, GCR, Docker Hub, or Harbor for module distribution.

## OCI Module Source Syntax

```hcl
module "vpc" {
  source = "oci://registry.example.com/terraform-modules/vpc:v2.1.0"

  cidr_block  = "10.0.0.0/16"
  environment = var.environment
}
```

## Using Public OCI Registries

```hcl
# Docker Hub
module "example" {
  source = "oci://docker.io/myorg/terraform-vpc:v1.0.0"
}

# GitHub Container Registry
module "vpc" {
  source = "oci://ghcr.io/myorg/terraform-vpc:v1.0.0"
}

# AWS ECR
module "vpc" {
  source = "oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/terraform-modules/vpc:v1.0.0"
}
```

## Authentication

```bash
# Docker Hub
docker login

# AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Google Artifact Registry
gcloud auth configure-docker us-docker.pkg.dev

# GitHub Container Registry
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
```

## Publishing a Module to OCI Registry

```bash
# Install oras (OCI Registry AS Storage)
brew install oras

# Package your module
tar -czf vpc-module.tar.gz ./modules/vpc/

# Push to OCI registry
oras push registry.example.com/terraform-modules/vpc:v2.1.0 \
  vpc-module.tar.gz:application/vnd.opentofu.module.v1+tar+gzip

# Tag as latest
oras tag registry.example.com/terraform-modules/vpc:v2.1.0 \
  registry.example.com/terraform-modules/vpc:latest
```

## Complete Example

```hcl
terraform {
  required_version = ">= 1.7"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

module "vpc" {
  source = "oci://ghcr.io/mycompany/terraform-aws-vpc:v2.0.0"

  vpc_cidr            = "10.0.0.0/16"
  availability_zones  = ["us-east-1a", "us-east-1b", "us-east-1c"]
  environment         = var.environment
}

module "eks" {
  source = "oci://ghcr.io/mycompany/terraform-aws-eks:v1.5.0"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
}
```

## Conclusion

OCI module sources let organizations reuse existing container registry infrastructure for Terraform/OpenTofu modules. This is particularly valuable for organizations already using ECR, GCR, Harbor, or Docker Hub, as it eliminates the need for a separate module registry while providing familiar tooling, access control, and scanning capabilities.
