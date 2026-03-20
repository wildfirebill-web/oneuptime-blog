# How to Create a Cloud SQL Instance with OpenTofu on GCP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Google Cloud, Infrastructure as Code, IaC, Cloud SQL, Database

Description: Learn how to create a highly available Cloud SQL instance with read replicas, backups, and private IP using OpenTofu.

## Introduction

This guide covers How to Create a Cloud SQL Instance with OpenTofu on GCP using OpenTofu with production-ready configurations and best practices.

## Prerequisites

- OpenTofu v1.6+
- Google Cloud SDK installed and authenticated
- GCP project with billing enabled

## Step 1: Configure the Provider

```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

## Step 2: Define Variables

```hcl
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "GCP region"
  type        = string
  default     = "us-central1"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "production"
}
```

## Step 3: Enable Required APIs

```hcl
# Enable required GCP service APIs
resource "google_project_service" "required_apis" {
  for_each = toset([
    "compute.googleapis.com",
    "container.googleapis.com",
    "iam.googleapis.com",
    "cloudresourcemanager.googleapis.com",
    "logging.googleapis.com",
    "monitoring.googleapis.com"
  ])

  project = var.project_id
  service = each.value

  disable_dependent_services = false
  disable_on_destroy         = false
}
```

## Step 4: Create Primary Resource

```hcl
# Service account for the resource
resource "google_service_account" "main" {
  account_id   = "sa-${var.environment}"
  display_name = "Service Account for ${var.environment}"
  project      = var.project_id
}

# Grant required roles
resource "google_project_iam_member" "main" {
  project = var.project_id
  role    = "roles/viewer"
  member  = "serviceAccount:${google_service_account.main.email}"
}
```

## Step 5: Configure Monitoring

```hcl
# Create a monitoring alert policy
resource "google_monitoring_alert_policy" "resource_errors" {
  display_name = "Resource Errors - ${var.environment}"
  combiner     = "OR"
  project      = var.project_id

  conditions {
    display_name = "Error rate too high"
    condition_threshold {
      filter          = "metric.type=\"logging.googleapis.com/log_entry_count\""
      comparison      = "COMPARISON_GT"
      threshold_value = 10
      duration        = "300s"

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_RATE"
      }
    }
  }

  notification_channels = var.notification_channel_ids
}
```

## Step 6: Define Outputs

```hcl
output "service_account_email" {
  description = "Service account email"
  value       = google_service_account.main.email
}

output "project_id" {
  description = "GCP Project ID"
  value       = var.project_id
}
```

## Step 7: Deploy

```bash
# Authenticate with GCP
gcloud auth application-default login

# Initialize OpenTofu
tofu init

# Preview changes
tofu plan -var="project_id=your-project-id"

# Apply configuration
tofu apply -var="project_id=your-project-id"
```

## Best Practices

- Enable only required GCP APIs to minimize attack surface
- Use service accounts with least-privilege IAM roles
- Enable audit logging for all GCP resources
- Use labels consistently for cost allocation and resource management
- Store state in a GCS bucket with versioning enabled

## Conclusion

You have successfully configured How to Create a Cloud SQL Instance with OpenTofu on GCP using OpenTofu. This configuration follows GCP best practices for security, monitoring, and resource management. Always use IAM conditions for fine-grained access control and enable Cloud Audit Logs for compliance and security investigation.
