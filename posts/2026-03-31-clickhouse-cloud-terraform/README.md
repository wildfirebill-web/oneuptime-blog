# How to Set Up ClickHouse Cloud with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Cloud, Terraform, Infrastructure as Code, DevOps, Automation

Description: Learn how to provision and manage ClickHouse Cloud services using Terraform with the official ClickHouse provider for repeatable infrastructure deployments.

---

The official ClickHouse Terraform provider lets you manage ClickHouse Cloud resources as code - creating services, configuring IP access lists, managing users, and setting up private endpoints through declarative configuration files.

## Installing the Provider

In your `versions.tf`:

```hcl
terraform {
  required_providers {
    clickhouse = {
      source  = "ClickHouse/clickhouse"
      version = "~> 1.0"
    }
  }
}
```

## Configuring Authentication

```hcl
provider "clickhouse" {
  organization_id = var.clickhouse_org_id
  token_key       = var.clickhouse_api_key
  token_secret    = var.clickhouse_api_secret
}
```

Store credentials in environment variables or a secrets manager - never hardcode them.

## Creating a ClickHouse Cloud Service

```hcl
resource "clickhouse_service" "analytics" {
  name        = "analytics-prod"
  cloud_provider = "aws"
  region      = "us-east-1"
  tier        = "production"

  min_total_memory_gb = 24
  max_total_memory_gb = 96

  ip_access = [
    {
      source      = "10.0.0.0/8"
      description = "Internal VPC"
    }
  ]
}

output "service_host" {
  value     = clickhouse_service.analytics.endpoints[0].host
  sensitive = true
}
```

## Managing Users

```hcl
resource "clickhouse_service_user" "analyst" {
  service_id = clickhouse_service.analytics.id
  name       = "analyst"
  password   = var.analyst_password
}
```

## Configuring Private Endpoints

```hcl
resource "clickhouse_service_private_endpoint_registration" "vpc" {
  cloud_provider = "aws"
  region         = "us-east-1"
  id             = aws_vpc_endpoint.clickhouse.id
  description    = "Production VPC endpoint"
}
```

## Plan and Apply

```bash
terraform init
terraform plan
terraform apply
```

## Importing Existing Services

If you have existing ClickHouse Cloud services, import them into Terraform state:

```bash
terraform import clickhouse_service.analytics <service-id>
```

## State Management Best Practices

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "clickhouse/prod.tfstate"
    region = "us-east-1"
  }
}
```

Store Terraform state remotely to enable team collaboration and prevent state conflicts.

## Summary

The ClickHouse Terraform provider enables full infrastructure-as-code management of ClickHouse Cloud resources. Define services, users, and networking in HCL, use remote state for team environments, and integrate with your existing CI/CD pipelines for automated deployments.
