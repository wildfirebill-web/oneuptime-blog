# How to Manage Multi-Cloud State Backends with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, State Management, Multi-Cloud, AWS, Azure, GCP, Infrastructure as Code

Description: Learn how to structure OpenTofu state backends for multi-cloud deployments - choosing the right backend per cloud, splitting state by cloud boundary, and enabling cross-cloud references via remote...

## Introduction

Multi-cloud OpenTofu deployments need a clear state strategy. The two main patterns are: a single state file managing all clouds together (simple but large blast radius), or separate state files per cloud (more isolation, cross-cloud references via `terraform_remote_state`).

## Option 1: Single S3 Backend for All Clouds

```hcl
# backend.tf - single backend for multi-cloud config

terraform {
  backend "s3" {
    bucket         = "my-company-tofu-state"
    key            = "multi-cloud/production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/abc123"
    dynamodb_table = "terraform-state-locks"
  }
}
```

Simple, but the state file grows large and every change to any cloud requires locking the whole state.

## Option 2: Separate State per Cloud (Recommended)

```text
environments/production/
  aws/
    backend.tf   # S3 backend
    main.tf
    outputs.tf   # VPC ID, subnet IDs
  azure/
    backend.tf   # Azure Storage backend
    main.tf
    outputs.tf   # VNet ID, subnet IDs
  gcp/
    backend.tf   # GCS backend
    main.tf
    outputs.tf   # VPC ID, subnet IDs
```

AWS state on S3:

```hcl
# environments/production/aws/backend.tf
terraform {
  backend "s3" {
    bucket         = "my-company-tofu-state"
    key            = "production/aws/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}
```

Azure state on Azure Storage:

```hcl
# environments/production/azure/backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "tofu-state-rg"
    storage_account_name = "mycompanytofustate"
    container_name       = "tfstate"
    key                  = "production/azure/terraform.tfstate"
  }
}
```

GCP state on GCS:

```hcl
# environments/production/gcp/backend.tf
terraform {
  backend "gcs" {
    bucket = "my-company-tofu-state"
    prefix = "production/gcp"
  }
}
```

## Cross-Cloud References via Remote State

```hcl
# environments/production/gcp/main.tf
# Reference AWS outputs from GCP config
data "terraform_remote_state" "aws" {
  backend = "s3"
  config = {
    bucket = "my-company-tofu-state"
    key    = "production/aws/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use AWS VPC CIDR in GCP firewall rule
resource "google_compute_firewall" "allow_aws" {
  name    = "allow-from-aws"
  network = google_compute_network.main.id

  allow {
    protocol = "tcp"
    ports    = ["443", "8080"]
  }

  # Reference the AWS VPC CIDR output
  source_ranges = [data.terraform_remote_state.aws.outputs.vpc_cidr]
}
```

## Option 3: Single Backend, Multiple Keys (Workspace Pattern)

```hcl
# Use S3 backend with per-cloud keys
terraform {
  backend "s3" {
    bucket         = "my-company-tofu-state"
    key            = "production/${var.cloud}/terraform.tfstate"  # ERROR: variables not allowed in backend
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"
  }
}
```

Variables are not allowed in backend configs. Instead, use `-backend-config`:

```bash
# Deploy AWS
tofu -chdir=environments/production/aws init \
  -backend-config="key=production/aws/terraform.tfstate"
tofu -chdir=environments/production/aws apply

# Deploy Azure
tofu -chdir=environments/production/azure init
tofu -chdir=environments/production/azure apply

# Deploy GCP
tofu -chdir=environments/production/gcp init
tofu -chdir=environments/production/gcp apply
```

## CI/CD: Parallelizing Multi-Cloud Deployments

```yaml
# .github/workflows/deploy-multi-cloud.yml
jobs:
  deploy-aws:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: us-east-1
      - run: tofu -chdir=environments/production/aws apply -auto-approve

  deploy-azure:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      - run: tofu -chdir=environments/production/azure apply -auto-approve

  deploy-gcp:
    needs: [deploy-aws]   # GCP depends on AWS outputs
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WIF_PROVIDER }}
          service_account: ${{ vars.GCP_SA_EMAIL }}
      - run: tofu -chdir=environments/production/gcp apply -auto-approve
```

## Conclusion

For multi-cloud OpenTofu deployments, use separate state files per cloud - each stored in its own cloud's native backend. Use `terraform_remote_state` data sources to reference outputs across cloud boundaries. Run cloud deployments in parallel in CI where there are no cross-cloud dependencies, and sequence them where outputs from one cloud are inputs to another.
