# How to Use Multi-Cloud State Backends in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, State Management, Multi-Cloud, AWS S3, Azure Blob, GCS

Description: Learn how to configure and manage OpenTofu state backends across multiple cloud providers, including AWS S3, Azure Blob Storage, and Google Cloud Storage.

## Introduction

In multi-cloud environments, different teams or infrastructure components may reside on different cloud providers. OpenTofu supports multiple backend types — AWS S3, Azure Blob Storage, GCS, and more — allowing you to store state close to the infrastructure it manages.

## AWS S3 Backend

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-tofu-state"
    key            = "aws/production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tofu-state-locks"
  }
}
```

Create supporting resources:

```hcl
resource "aws_s3_bucket" "state" {
  bucket = "mycompany-tofu-state"
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_dynamodb_table" "lock" {
  name         = "tofu-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## Azure Blob Storage Backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tofu-state-rg"
    storage_account_name = "mycompanytofustate"
    container_name       = "tfstate"
    key                  = "azure/production/terraform.tfstate"
    use_oidc             = true
  }
}
```

Create the storage account:

```bash
az group create --name tofu-state-rg --location eastus

az storage account create \
  --name mycompanytofustate \
  --resource-group tofu-state-rg \
  --sku Standard_LRS \
  --encryption-services blob \
  --allow-blob-public-access false

az storage container create \
  --name tfstate \
  --account-name mycompanytofustate
```

## Google Cloud Storage Backend

```hcl
terraform {
  backend "gcs" {
    bucket = "mycompany-tofu-state"
    prefix = "gcp/production"
  }
}
```

Create the GCS bucket:

```bash
gsutil mb -l US-EAST1 gs://mycompany-tofu-state
gsutil versioning set on gs://mycompany-tofu-state
gsutil uniformbucketlevelaccess set on gs://mycompany-tofu-state
```

## Organizing Multi-Cloud State

Use a consistent naming convention for state keys:

```
{cloud}/{region}/{environment}/{component}/terraform.tfstate
```

Examples:
- `aws/us-east-1/production/networking/terraform.tfstate`
- `azure/eastus/production/aks-cluster/terraform.tfstate`
- `gcp/us-east1/production/gke-cluster/terraform.tfstate`

## Reading Cross-Cloud State with Remote State Data Sources

Reference state from another cloud's backend:

```hcl
# In GCP config, reference networking outputs from AWS config
data "terraform_remote_state" "aws_networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-tofu-state"
    key    = "aws/us-east-1/production/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use the output
resource "google_compute_vpn_gateway" "to_aws" {
  name    = "to-aws"
  network = google_compute_network.main.id

  # Reference AWS VPN endpoint from remote state
  # target_vpn_gateway = data.terraform_remote_state.aws_networking.outputs.vpn_endpoint
}
```

## Partial Backend Configuration

Use partial configuration for flexibility across environments:

```hcl
# backend.tf (no values hardcoded)
terraform {
  backend "s3" {}
}
```

Pass values at init time:

```bash
tofu init \
  -backend-config="bucket=mycompany-tofu-state" \
  -backend-config="key=aws/production/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="encrypt=true" \
  -backend-config="dynamodb_table=tofu-state-locks"
```

Or use a backend config file per environment:

```hcl
# configs/production.s3.tfbackend
bucket         = "mycompany-tofu-state"
key            = "production/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
dynamodb_table = "tofu-state-locks"
```

```bash
tofu init -backend-config=configs/production.s3.tfbackend
```

## Best Practices

- Use one backend bucket per cloud account and environment for isolation.
- Enable versioning on all state buckets for rollback capability.
- Encrypt state at rest using cloud-native encryption.
- Use OIDC/workload identity federation instead of long-lived credentials for CI/CD.
- Restrict state bucket access with IAM policies — state files contain sensitive data.
- Document the state key structure so teams can easily locate their state.

## Conclusion

OpenTofu's support for multiple backend types makes it well-suited for multi-cloud environments. By standardizing on a consistent key naming convention and using partial backend configurations, you can manage infrastructure across AWS, Azure, and GCP with a unified workflow.
