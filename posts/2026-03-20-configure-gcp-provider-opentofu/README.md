# How to Configure the Google Cloud Provider in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Provider Configuration, Infrastructure as Code, Google Cloud

Description: Learn how to configure the Google Cloud provider in OpenTofu with project settings, credentials, and regional defaults for managing GCP resources.

## Introduction

The Google Cloud provider (hashicorp/google) enables OpenTofu to manage GCP resources. Configuration covers project selection, region/zone defaults, and authentication—which can use service account keys, Workload Identity, or Application Default Credentials.

## Minimal Configuration

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }
  required_version = ">= 1.6.0"
}

provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}
```

## Authentication Methods

The provider checks credentials in this order:
1. `credentials` argument in the provider block
2. `GOOGLE_CREDENTIALS` environment variable
3. Application Default Credentials (`gcloud auth application-default login`)
4. Metadata server (on GCE instances)

For CI/CD, use the environment variable:

```bash
export GOOGLE_CREDENTIALS=$(cat /path/to/service-account-key.json)
export GOOGLE_PROJECT="my-project-id"
```

## Full Production Configuration

```hcl
provider "google" {
  project = var.project_id
  region  = var.region

  default_labels = {
    managed_by  = "opentofu"
    environment = var.environment
    team        = var.team
  }
}

# Beta resources require the google-beta provider
provider "google-beta" {
  project = var.project_id
  region  = var.region
}
```

## Variables

```hcl
variable "project_id"  { type = string }
variable "region"      { type = string; default = "us-central1" }
variable "zone"        { type = string; default = "us-central1-a" }
variable "environment" { type = string }
variable "team"        { type = string }
```

## Multi-Project Setup

```hcl
provider "google" {
  alias   = "project_a"
  project = var.project_a_id
  region  = "us-central1"
}

provider "google" {
  alias   = "project_b"
  project = var.project_b_id
  region  = "europe-west1"
}

resource "google_storage_bucket" "logs" {
  provider = google.project_a
  name     = "logs-${var.project_a_id}"
  location = "US"
}
```

## Conclusion

The GCP provider's `default_labels` block ensures consistent labelling across all resources without repeating label blocks. Always pin your provider version in the `required_providers` block and commit the `.terraform.lock.hcl` file to lock exact provider versions for your team.
