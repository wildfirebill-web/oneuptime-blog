# How to Manage GCP Committed Use Discounts with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Committed Use Discounts, Cost Optimization, Infrastructure as Code

Description: Learn how to purchase and manage GCP Committed Use Discounts with OpenTofu for up to 57% savings on Compute Engine and Cloud SQL workloads.

GCP Committed Use Discounts (CUDs) provide significant discounts in exchange for 1 or 3-year commitments to a minimum level of vCPUs, memory, and other resources. Managing commitments in OpenTofu keeps your cost optimization strategy documented.

## Provider Configuration

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = "us-central1"
}
```

## Resource-Based Commitment (Compute Engine)

```hcl
resource "google_compute_region_commitment" "production" {
  name   = "production-commitment"
  region = "us-central1"
  plan   = "THIRTY_SIX_MONTH"  # TWELVE_MONTH or THIRTY_SIX_MONTH

  resources {
    type   = "VCPU"
    amount = "32"  # Commit to 32 vCPUs
  }

  resources {
    type   = "MEMORY"
    amount = "131072"  # 128 GB RAM in MB
  }
}
```

## Machine Type Commitment

```hcl
resource "google_compute_region_commitment" "n2_standard" {
  name   = "n2-standard-commitment"
  region = "us-central1"
  plan   = "TWELVE_MONTH"

  resources {
    type              = "N2_CPUS"
    amount            = "16"
    accelerator_type  = ""
  }
}
```

## GPU Commitment

```hcl
resource "google_compute_region_commitment" "gpu_commitment" {
  name   = "gpu-commitment"
  region = "us-central1"
  plan   = "TWELVE_MONTH"

  resources {
    type             = "NVIDIA_A100_GPUS"
    amount           = "4"
    accelerator_type = "nvidia-tesla-a100"
  }
}
```

## Cloud SQL Commitment

GCP Cloud SQL committed use discounts are automatic based on consistent usage, not resource-purchased commitments like Compute Engine. However, you can track your Cloud SQL spend and set budgets:

```hcl
resource "google_billing_budget" "cloud_sql" {
  billing_account = var.billing_account_id
  display_name    = "Cloud SQL Monthly Budget"

  budget_filter {
    projects = ["projects/${var.project_id}"]
    services = ["services/95FF-2EF5-5EA1"]  # Cloud SQL service ID
  }

  amount {
    specified_amount {
      currency_code = "USD"
      units         = "5000"
    }
  }

  threshold_rules {
    threshold_percent = 0.8
  }

  threshold_rules {
    threshold_percent = 1.0
  }
}
```

## Viewing Commitments and Utilization

```bash
# List active commitments
gcloud compute commitments list --region=us-central1

# Check commitment utilization
gcloud compute commitments describe production-commitment \
  --region=us-central1 \
  --format="json" | jq '.resources[].amount, .resources[].acceleratorType'
```

## Budget Alert for Committed Use Savings

```hcl
resource "google_billing_budget" "compute_budget" {
  billing_account = var.billing_account_id
  display_name    = "Compute Engine Monthly Budget"

  budget_filter {
    projects  = ["projects/${var.project_id}"]
    services  = ["services/6F81-5844-456A"]  # Compute Engine service ID
    credit_types_treatment = "INCLUDE_SPECIFIED_CREDITS"
    credit_types = ["COMMITTED_USAGE_DISCOUNT_PROGRAM"]
  }

  amount {
    specified_amount {
      currency_code = "USD"
      units         = "20000"
    }
  }

  threshold_rules {
    threshold_percent = 0.9
    spend_basis       = "CURRENT_SPEND"
  }

  threshold_rules {
    threshold_percent = 1.1
    spend_basis       = "FORECASTED_SPEND"
  }

  all_updates_rule {
    pubsub_topic                     = google_pubsub_topic.budget_alerts.id
    schema_version                   = "1.0"
    monitoring_notification_channels = [google_monitoring_notification_channel.email.name]
  }
}
```

## Conclusion

GCP Committed Use Discounts in OpenTofu provide up to 57% savings for predictable Compute Engine workloads. Purchase resource-based commitments for flexible savings across all machine types, or machine-type commitments for the highest discount on specific families. Monitor commitment utilization through the GCP console and set billing budgets to track savings and prevent overspend.
