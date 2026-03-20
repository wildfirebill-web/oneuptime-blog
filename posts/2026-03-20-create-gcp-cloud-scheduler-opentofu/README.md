# How to Create GCP Cloud Scheduler Jobs with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Cloud Scheduler, Cron Jobs, Infrastructure as Code

Description: Learn how to create GCP Cloud Scheduler jobs with OpenTofu to run cron-like scheduled tasks that invoke HTTP endpoints, Pub/Sub topics, and App Engine services.

GCP Cloud Scheduler is a managed cron service that reliably invokes targets on a schedule. Managing scheduler jobs in OpenTofu ensures your scheduled tasks are documented, version-controlled, and consistently deployed across environments.

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

## HTTP Target Job

```hcl
resource "google_cloud_scheduler_job" "daily_cleanup" {
  name        = "daily-cleanup"
  description = "Run database cleanup at 2 AM UTC daily"
  schedule    = "0 2 * * *"  # Cron expression
  time_zone   = "UTC"
  region      = "us-central1"

  http_target {
    http_method = "POST"
    uri         = "https://myapp.example.com/internal/cleanup"

    headers = {
      "Content-Type" = "application/json"
    }

    body = base64encode(jsonencode({
      action    = "cleanup"
      dry_run   = false
    }))

    oidc_token {
      service_account_email = google_service_account.scheduler.email
      audience              = "https://myapp.example.com"
    }
  }

  retry_config {
    retry_count          = 3
    max_retry_duration   = "0s"  # No max duration
    min_backoff_duration = "5s"
    max_backoff_duration = "60s"
    max_doublings        = 3
  }

  attempt_deadline = "320s"  # Job attempt timeout
}
```

## Pub/Sub Target Job

```hcl
resource "google_pubsub_topic" "scheduled_tasks" {
  name = "scheduled-tasks"
}

resource "google_cloud_scheduler_job" "report_generator" {
  name        = "weekly-report-generator"
  description = "Generate weekly business report every Monday 9 AM"
  schedule    = "0 9 * * 1"  # Monday 9 AM
  time_zone   = "America/New_York"
  region      = "us-central1"

  pubsub_target {
    topic_name = google_pubsub_topic.scheduled_tasks.id

    data = base64encode(jsonencode({
      task   = "generate_weekly_report"
      report = "business_summary"
    }))

    attributes = {
      task_type = "report"
      priority  = "normal"
    }
  }
}
```

## Cloud Run Target

```hcl
resource "google_cloud_scheduler_job" "sync_job" {
  name      = "data-sync-hourly"
  schedule  = "0 * * * *"  # Every hour
  time_zone = "UTC"
  region    = "us-central1"

  http_target {
    http_method = "POST"
    uri         = "${google_cloud_run_service.worker.status[0].url}/sync"

    headers = {
      "Content-Type" = "application/json"
    }

    body = base64encode("{\"action\":\"sync\"}")

    # Authenticate using OIDC (Cloud Run requires authentication)
    oidc_token {
      service_account_email = google_service_account.scheduler.email
      audience              = google_cloud_run_service.worker.status[0].url
    }
  }
}
```

## Service Account for Scheduler

```hcl
resource "google_service_account" "scheduler" {
  account_id   = "cloud-scheduler"
  display_name = "Cloud Scheduler Service Account"
}

# Allow scheduler to invoke Cloud Run
resource "google_cloud_run_service_iam_member" "scheduler_invoker" {
  service  = google_cloud_run_service.worker.name
  location = google_cloud_run_service.worker.location
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.scheduler.email}"
}

# Allow scheduler to publish to Pub/Sub
resource "google_project_iam_member" "scheduler_pubsub" {
  project = var.project_id
  role    = "roles/pubsub.publisher"
  member  = "serviceAccount:${google_service_account.scheduler.email}"
}
```

## Multiple Jobs with for_each

```hcl
locals {
  jobs = {
    cleanup   = { schedule = "0 2 * * *",   description = "Nightly cleanup" }
    report    = { schedule = "0 9 * * 1",   description = "Weekly report" }
    backup    = { schedule = "0 3 * * *",   description = "Daily backup" }
    heartbeat = { schedule = "*/5 * * * *", description = "5-min health check" }
  }
}

resource "google_cloud_scheduler_job" "jobs" {
  for_each    = local.jobs
  name        = each.key
  description = each.value.description
  schedule    = each.value.schedule
  time_zone   = "UTC"
  region      = "us-central1"

  http_target {
    http_method = "POST"
    uri         = "https://myapp.example.com/tasks/${each.key}"
    oidc_token {
      service_account_email = google_service_account.scheduler.email
    }
  }
}
```

## Conclusion

GCP Cloud Scheduler jobs in OpenTofu give you version-controlled, reliable cron scheduling. Use OIDC authentication for Cloud Run and App Engine targets, create service accounts with minimal permissions, and define all jobs in a single for_each block for easy maintenance. Set retry policies to handle transient failures without manual intervention.
