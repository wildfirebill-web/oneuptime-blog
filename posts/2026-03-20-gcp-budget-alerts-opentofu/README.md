# How to Create GCP Budget Alerts with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Billing, Budget Alerts, Infrastructure as Code

Description: Learn how to create GCP billing budgets and alerts with OpenTofu to monitor and control Google Cloud spending across projects and services.

GCP Billing Budgets alert your team when spending exceeds thresholds and can automatically disable billing when critical limits are reached. Managing budgets in OpenTofu ensures all projects have cost guardrails from day one.

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
  project         = var.project_id
  billing_project = var.project_id
}
```

## Project Budget

```hcl
resource "google_billing_budget" "project_budget" {
  billing_account = var.billing_account_id
  display_name    = "Production Project Budget"

  # Filter to a specific project
  budget_filter {
    projects = ["projects/${var.project_id}"]
  }

  amount {
    specified_amount {
      currency_code = "USD"
      units         = "10000"  # $10,000/month
    }
  }

  # Alert at 50%
  threshold_rules {
    threshold_percent = 0.5
    spend_basis       = "CURRENT_SPEND"
  }

  # Alert at 90%
  threshold_rules {
    threshold_percent = 0.9
    spend_basis       = "CURRENT_SPEND"
  }

  # Alert at 100%
  threshold_rules {
    threshold_percent = 1.0
    spend_basis       = "CURRENT_SPEND"
  }

  # Alert when forecasted to exceed 110%
  threshold_rules {
    threshold_percent = 1.1
    spend_basis       = "FORECASTED_SPEND"
  }

  all_updates_rule {
    pubsub_topic  = google_pubsub_topic.budget_alerts.id
    schema_version = "1.0"

    monitoring_notification_channels = [
      google_monitoring_notification_channel.email.name,
      google_monitoring_notification_channel.slack.name,
    ]

    # Automatically disable billing at 100% (prevents runaway spend)
    disable_default_iam_recipients = false
  }
}
```

## Email Notification Channel

```hcl
resource "google_monitoring_notification_channel" "email" {
  display_name = "Engineering FinOps Email"
  type         = "email"

  labels = {
    email_address = "finops@example.com"
  }
}

resource "google_monitoring_notification_channel" "slack" {
  display_name = "FinOps Slack Alerts"
  type         = "slack"

  labels = {
    channel_name = "#finops-alerts"
  }

  sensitive_labels {
    auth_token = var.slack_token
  }
}
```

## Pub/Sub Budget Alert Handler

```hcl
resource "google_pubsub_topic" "budget_alerts" {
  name = "billing-budget-alerts"
}

resource "google_pubsub_subscription" "budget_handler" {
  name  = "budget-alert-handler"
  topic = google_pubsub_topic.budget_alerts.id

  push_config {
    push_endpoint = var.budget_handler_url
  }
}
```

## Service-Specific Budget

```hcl
resource "google_billing_budget" "bigquery_budget" {
  billing_account = var.billing_account_id
  display_name    = "BigQuery Monthly Budget"

  budget_filter {
    projects = ["projects/${var.project_id}"]
    services = ["services/95FF-2EF5-5EA1"]  # BigQuery service ID
  }

  amount {
    specified_amount {
      currency_code = "USD"
      units         = "2000"
    }
  }

  threshold_rules {
    threshold_percent = 0.8
  }

  threshold_rules {
    threshold_percent = 1.0
  }

  all_updates_rule {
    monitoring_notification_channels = [google_monitoring_notification_channel.email.name]
    pubsub_topic = google_pubsub_topic.budget_alerts.id
  }
}
```

## Multiple Project Budgets

```hcl
locals {
  project_budgets = {
    "production"  = { project = var.prod_project_id,  amount = 50000 }
    "staging"     = { project = var.staging_project_id, amount = 5000 }
    "development" = { project = var.dev_project_id,   amount = 2000 }
  }
}

resource "google_billing_budget" "projects" {
  for_each = local.project_budgets

  billing_account = var.billing_account_id
  display_name    = "${each.key} project budget"

  budget_filter {
    projects = ["projects/${each.value.project}"]
  }

  amount {
    specified_amount {
      currency_code = "USD"
      units         = tostring(each.value.amount)
    }
  }

  threshold_rules {
    threshold_percent = 0.9
  }

  threshold_rules {
    threshold_percent = 1.0
  }

  all_updates_rule {
    monitoring_notification_channels = [google_monitoring_notification_channel.email.name]
  }
}
```

## Conclusion

GCP Billing Budgets in OpenTofu provide automated spending visibility across all your projects. Route budget alerts to both email notification channels and Pub/Sub topics for programmatic handling (e.g., automatic project shutdown). Set forecasted alerts to catch trends before the budget is breached, and use service-specific budgets to identify which GCP services are driving unexpected costs.
