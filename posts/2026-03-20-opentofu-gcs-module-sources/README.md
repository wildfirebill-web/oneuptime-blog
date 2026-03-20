# How to Use GCS Bucket Module Sources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Modules, GCP

Description: Learn how to use Google Cloud Storage bucket module sources in OpenTofu to host and distribute private modules on GCP.

## Introduction

OpenTofu supports Google Cloud Storage (GCS) as a module source. This allows GCP users to distribute private infrastructure modules using their existing GCS infrastructure.

## Syntax

```hcl
module "example" {
  source = "gcs::https://www.googleapis.com/storage/v1/BUCKET_NAME/PATH/TO/module.zip"
}
```

The `gcs::` prefix enables GCS-specific authentication using Application Default Credentials.

## Examples

### Basic GCS Module Source

```hcl
module "vpc" {
  source = "gcs::https://www.googleapis.com/storage/v1/mycompany-terraform-modules/vpc/vpc-v2.0.0.zip"

  project_id  = var.project_id
  region      = var.region
  cidr_block  = "10.0.0.0/16"
}
```

### Versioned Modules

```hcl
module "gke_cluster" {
  source = "gcs::https://www.googleapis.com/storage/v1/mycompany-modules/gke/v1.5.0/cluster.zip"

  project_id   = var.project_id
  cluster_name = "prod-cluster"
  region       = "us-central1"
}
```

## Publishing Modules to GCS

### Step 1: Create the GCS Bucket

```hcl
resource "google_storage_bucket" "modules" {
  name          = "mycompany-terraform-modules"
  location      = "US"
  force_destroy = false

  versioning {
    enabled = true
  }

  uniform_bucket_level_access = true
}
```

### Step 2: Upload Module Archives

```bash
# Package the module
cd modules/vpc
zip -r /tmp/vpc-v1.0.0.zip .

# Upload to GCS
gsutil cp /tmp/vpc-v1.0.0.zip gs://mycompany-terraform-modules/vpc/vpc-v1.0.0.zip
```

### Step 3: Grant Access

```hcl
resource "google_storage_bucket_iam_member" "module_reader" {
  bucket = google_storage_bucket.modules.name
  role   = "roles/storage.objectViewer"
  member = "serviceAccount:terraform@my-project.iam.gserviceaccount.com"
}
```

## Authentication

OpenTofu uses Google Application Default Credentials (ADC):

```bash
# Authenticate locally
gcloud auth application-default login

# Or use service account
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
```

## Step-by-Step Usage

1. Create the GCS bucket.
2. Package and upload module archives.
3. Grant IAM access to the Terraform service account.
4. Reference with `gcs::` prefix.
5. Run `tofu init`.

## Conclusion

GCS bucket module sources provide a natural module distribution mechanism for GCP-based infrastructure teams. Combined with GCS versioning and IAM access control, it offers a secure, scalable way to distribute private OpenTofu modules within your GCP organization.
