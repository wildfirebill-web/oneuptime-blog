---
title: "What Providers Do in OpenTofu"
author: nawazdhandala
tags: opentofu, terraform, iac, providers
description: "Understand what providers are in OpenTofu, how they work, and why they are central to managing infrastructure."
---

# What Providers Do in OpenTofu

Providers are the plugins that allow OpenTofu to interact with external services — cloud platforms, SaaS products, APIs, and more. Every resource in your configuration is managed by a provider, which handles authentication and API communication.

## The Provider Architecture

OpenTofu itself knows nothing about AWS, GCP, Azure, or any other service. Providers are separately compiled plugins that extend OpenTofu's capabilities:

```
OpenTofu Core
     |
     |-- hashicorp/aws    (manages EC2, S3, RDS, etc.)
     |-- hashicorp/google (manages GCE, GKE, CloudSQL, etc.)
     |-- hashicorp/azurerm (manages VMs, AKS, etc.)
     |-- hashicorp/kubernetes (manages pods, deployments, etc.)
     |-- mycorp/internal  (manages internal systems)
```

## Declaring a Provider

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

## What Providers Provide

Each provider exposes:

1. **Resources** — Infrastructure objects you create, update, and delete
2. **Data sources** — Read-only queries of existing infrastructure
3. **Functions** — Provider-defined functions (OpenTofu 1.7+)

```hcl
# Resource — creates and manages an EC2 instance
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
}

# Data source — reads an existing VPC
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# Provider function (OpenTofu 1.7+)
output "arn_region" {
  value = provider::aws::arn_parse(aws_instance.web.arn).region
}
```

## Provider Initialization

When you run `tofu init`, OpenTofu:

1. Reads `required_providers` declarations
2. Finds matching providers in the registry
3. Downloads the correct version
4. Verifies checksums against `.terraform.lock.hcl`
5. Installs them in `.terraform/providers/`

```bash
tofu init

# Output:
# Initializing provider plugins...
# - Finding hashicorp/aws versions matching "~> 5.0"...
# - Installing hashicorp/aws v5.38.0...
# - Installed hashicorp/aws v5.38.0 (signed by HashiCorp)
```

## Provider Authentication

Providers authenticate with external services using credentials you configure:

```hcl
# AWS — multiple authentication methods
provider "aws" {
  region = "us-east-1"

  # Option 1: explicit credentials (not recommended)
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key

  # Option 2: use shared credentials file (~/.aws/credentials)
  # Option 3: use environment variables (AWS_ACCESS_KEY_ID, etc.)
  # Option 4: use IAM roles (recommended for production)
  assume_role {
    role_arn = "arn:aws:iam::123456789012:role/TerraformRole"
  }
}

# Google Cloud
provider "google" {
  project     = "my-project"
  region      = "us-central1"
  credentials = file("service-account.json")
}

# Kubernetes
provider "kubernetes" {
  host                   = data.aws_eks_cluster.main.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.main.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.main.token
}
```

## Multiple Provider Configurations

You can configure the same provider multiple times using aliases:

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_instance" "east" {
  ami           = "ami-east"
  instance_type = "t3.medium"
}

resource "aws_instance" "west" {
  provider      = aws.west
  ami           = "ami-west"
  instance_type = "t3.medium"
}
```

## Provider Version Locking

The `.terraform.lock.hcl` file locks provider versions:

```hcl
# .terraform.lock.hcl (auto-generated)
provider "registry.opentofu.org/hashicorp/aws" {
  version     = "5.38.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abc123...",
    "zh:def456...",
  ]
}
```

Always commit this file to version control to ensure consistent provider versions across your team.

## Conclusion

Providers are the bridge between OpenTofu's declarative language and the external APIs that manage real infrastructure. Understanding how providers work — their authentication, versioning, and the resources/data sources they expose — is fundamental to working effectively with OpenTofu.
