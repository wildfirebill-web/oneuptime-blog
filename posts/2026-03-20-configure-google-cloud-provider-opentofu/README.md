# How to Configure the Google Cloud Provider in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Google Cloud, Infrastructure as Code, IaC, Provider Configuration

Description: Learn how to configure the Google Cloud provider in OpenTofu with project settings, credentials, and default resource configurations.

## Introduction

The Google Cloud provider enables OpenTofu to manage GCP resources. This guide covers provider configuration, authentication options, and project setup for Google Cloud deployments.

## Prerequisites

- OpenTofu v1.6+
- Google Cloud SDK installed
- A GCP project with billing enabled

## Step 1: Basic Provider Configuration

```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}
```

## Step 2: Multiple Providers for Multi-Region

```hcl
provider "google" {
  project = var.project_id
  region  = "us-central1"
  alias   = "us_central"
}

provider "google" {
  project = var.project_id
  region  = "europe-west1"
  alias   = "europe"
}

# Use specific region provider
resource "google_compute_network" "vpc_us" {
  provider = google.us_central
  name     = "vpc-us"
}

resource "google_compute_network" "vpc_eu" {
  provider = google.europe
  name     = "vpc-eu"
}
```

## Step 3: Define Variables

```hcl
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "Default GCP region"
  type        = string
  default     = "us-central1"
}

variable "zone" {
  description = "Default GCP zone"
  type        = string
  default     = "us-central1-a"
}
```

## Step 4: Enable Required APIs

```hcl
# Enable required Google Cloud APIs
resource "google_project_service" "compute" {
  project            = var.project_id
  service            = "compute.googleapis.com"
  disable_on_destroy = false
}

resource "google_project_service" "container" {
  project            = var.project_id
  service            = "container.googleapis.com"
  disable_on_destroy = false
}

resource "google_project_service" "iam" {
  project            = var.project_id
  service            = "iam.googleapis.com"
  disable_on_destroy = false
}
```

## Step 5: Set Default Labels

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
```

## Step 6: Deploy

```bash
# Authenticate with gcloud
gcloud auth application-default login

tofu init
tofu plan
tofu apply
```

## Conclusion

You have successfully configured the Google Cloud provider in OpenTofu with multi-region support, default labels, and API enablement. Enable only the GCP APIs you need to reduce the attack surface. Use the `google-beta` provider alias for features not yet in the GA provider.
