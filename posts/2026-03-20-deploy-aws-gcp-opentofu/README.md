# How to Deploy to Both AWS and GCP with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, GCP, Multi-Cloud, Infrastructure as Code, DevOps

Description: Learn how to use OpenTofu to provision resources across both AWS and Google Cloud Platform simultaneously - deploying a multi-cloud architecture from a single configuration.

## Introduction

AWS and GCP are often used together - for example, AWS for primary compute with GCP BigQuery for analytics, or GCP for data pipelines feeding AWS-hosted applications. OpenTofu manages both from a single configuration.

## Provider Configuration

```hcl
# providers.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
}
```

## Variables

```hcl
variable "aws_region"     { default = "us-east-1" }
variable "gcp_project_id" { type = string }
variable "gcp_region"     { default = "us-central1" }
variable "environment"    { default = "prod" }
variable "project"        { default = "analytics-platform" }
```

## AWS Application Tier + GCP Analytics

```hcl
# AWS: Application servers
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "${var.project}-vpc", Environment = var.environment }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.large"
  subnet_id     = aws_subnet.private.id

  tags = {
    Name        = "${var.project}-app"
    Environment = var.environment
  }
}

# AWS: S3 bucket for raw data
resource "aws_s3_bucket" "raw_data" {
  bucket = "${var.project}-raw-data-${var.environment}"
}

# GCP: BigQuery dataset for analytics
resource "google_bigquery_dataset" "analytics" {
  dataset_id  = replace("${var.project}_analytics_${var.environment}", "-", "_")
  location    = "US"
  description = "Analytics dataset for ${var.project}"

  labels = {
    environment = var.environment
    project     = replace(var.project, "-", "_")
  }
}

resource "google_bigquery_table" "events" {
  dataset_id = google_bigquery_dataset.analytics.dataset_id
  table_id   = "events"

  schema = jsonencode([
    { name = "event_id",   type = "STRING",    mode = "REQUIRED" },
    { name = "event_type", type = "STRING",    mode = "REQUIRED" },
    { name = "user_id",    type = "STRING",    mode = "NULLABLE" },
    { name = "timestamp",  type = "TIMESTAMP", mode = "REQUIRED" }
  ])
}

# GCP: Cloud Storage for processed data
resource "google_storage_bucket" "processed" {
  name     = "${var.project}-processed-${var.environment}"
  location = "US"

  labels = {
    environment = var.environment
  }
}
```

## Cross-Cloud Data Transfer: AWS to GCP

```hcl
# GCP: Service account for AWS-to-GCP data access
resource "google_service_account" "data_transfer" {
  account_id   = "${var.project}-data-transfer"
  display_name = "AWS to GCP Data Transfer"
}

resource "google_storage_bucket_iam_member" "data_transfer_write" {
  bucket = google_storage_bucket.processed.name
  role   = "roles/storage.objectCreator"
  member = "serviceAccount:${google_service_account.data_transfer.email}"
}

# AWS: IAM policy for S3 read access
data "aws_iam_policy_document" "data_transfer" {
  statement {
    actions   = ["s3:GetObject", "s3:ListBucket"]
    resources = [
      aws_s3_bucket.raw_data.arn,
      "${aws_s3_bucket.raw_data.arn}/*"
    ]
  }
}

resource "aws_iam_policy" "data_transfer" {
  name   = "${var.project}-data-transfer"
  policy = data.aws_iam_policy_document.data_transfer.json
}
```

## Outputs: Cross-Cloud References

```hcl
output "aws_app_instance_id" {
  value = aws_instance.app.id
}

output "aws_raw_data_bucket" {
  value = aws_s3_bucket.raw_data.id
}

output "gcp_bigquery_dataset" {
  value = google_bigquery_dataset.analytics.dataset_id
}

output "gcp_processed_bucket" {
  value = google_storage_bucket.processed.name
}

output "gcp_service_account_email" {
  value     = google_service_account.data_transfer.email
  sensitive = false
}
```

## CI/CD Authentication

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    aws-region: us-east-1

- name: Authenticate to GCP
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
    service_account: opentofu@my-project.iam.gserviceaccount.com

- name: OpenTofu Apply
  run: |
    tofu init
    tofu apply -auto-approve \
      -var="gcp_project_id=${{ vars.GCP_PROJECT_ID }}"
```

## Conclusion

OpenTofu handles AWS and GCP resources in the same run, making cross-cloud architectures manageable as code. Common patterns include AWS compute with GCP analytics (BigQuery, Dataflow), cross-cloud data pipelines, and GCP ML workloads consuming AWS-hosted data. Authenticate using OIDC/Workload Identity on both clouds for keyless CI/CD integration.
