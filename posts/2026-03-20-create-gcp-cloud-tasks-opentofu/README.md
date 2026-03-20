# How to Create GCP Cloud Tasks with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Cloud Tasks, Task Queue, Infrastructure as Code

Description: Learn how to create GCP Cloud Tasks queues with OpenTofu for reliable asynchronous task execution with rate limiting and retry control.

GCP Cloud Tasks lets you manage the execution, dispatch, and delivery of tasks. Queues provide rate limiting and retry policies for asynchronous work. Managing queues in OpenTofu ensures consistent configuration across environments.

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

## Creating a Task Queue

```hcl
resource "google_cloud_tasks_queue" "default" {
  name     = "default-queue"
  location = "us-central1"

  rate_limits {
    max_concurrent_dispatches = 100  # Max tasks dispatched at once
    max_dispatches_per_second = 500  # Max dispatch rate
  }

  retry_config {
    max_attempts       = 5
    max_retry_duration = "3600s"  # 1 hour max retry window
    min_backoff        = "10s"
    max_backoff        = "300s"
    max_doublings      = 3  # Max exponential doublings
  }
}
```

## Queue with App Engine Target

```hcl
resource "google_cloud_tasks_queue" "email" {
  name     = "email-queue"
  location = "us-central1"

  app_engine_routing_override {
    service = "worker"
    version = "prod"
  }

  rate_limits {
    max_concurrent_dispatches = 10  # Throttle email sending
    max_dispatches_per_second = 5
  }

  retry_config {
    max_attempts = 10
    min_backoff  = "30s"
    max_backoff  = "600s"
    max_doublings = 4
  }
}
```

## Queue with HTTP Target

```hcl
resource "google_cloud_tasks_queue" "http_worker" {
  name     = "http-worker-queue"
  location = "us-central1"

  rate_limits {
    max_concurrent_dispatches = 50
    max_dispatches_per_second = 100
  }

  retry_config {
    max_attempts = 3
    min_backoff  = "5s"
    max_backoff  = "60s"
    max_doublings = 2
  }

  # Stack driver logging
  stackdriver_logging_config {
    sampling_ratio = 0.01  # Log 1% of tasks for debugging
  }
}
```

## Multiple Queues for Different Priority Levels

```hcl
locals {
  queues = {
    critical = {
      max_concurrent = 200
      max_per_second = 1000
      max_attempts   = 3
    }
    standard = {
      max_concurrent = 50
      max_per_second = 200
      max_attempts   = 5
    }
    background = {
      max_concurrent = 10
      max_per_second = 20
      max_attempts   = 10
    }
  }
}

resource "google_cloud_tasks_queue" "queues" {
  for_each = local.queues

  name     = "${each.key}-queue"
  location = "us-central1"

  rate_limits {
    max_concurrent_dispatches = each.value.max_concurrent
    max_dispatches_per_second = each.value.max_per_second
  }

  retry_config {
    max_attempts = each.value.max_attempts
    min_backoff  = "5s"
    max_backoff  = "300s"
    max_doublings = 3
  }
}
```

## IAM Permissions

```hcl
# Allow Cloud Tasks to invoke Cloud Run service

resource "google_cloud_run_service_iam_member" "tasks_invoker" {
  service  = google_cloud_run_service.worker.name
  location = google_cloud_run_service.worker.location
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.tasks.email}"
}

resource "google_service_account" "tasks" {
  account_id   = "cloud-tasks-invoker"
  display_name = "Cloud Tasks Service Account"
}

# Allow application to enqueue tasks
resource "google_project_iam_member" "task_enqueuer" {
  project = var.project_id
  role    = "roles/cloudtasks.enqueuer"
  member  = "serviceAccount:${google_service_account.app.email}"
}
```

## Outputs

```hcl
output "queue_names" {
  value = {
    for k, v in google_cloud_tasks_queue.queues : k => v.name
  }
}
```

## Conclusion

GCP Cloud Tasks queues in OpenTofu provide reliable, rate-limited task execution for asynchronous workloads. Define rate limits to prevent overwhelming downstream services, configure retry policies with exponential backoff, and create multiple priority queues when different task types need different throughput guarantees. Grant the Cloud Tasks service account invoker permissions on your Cloud Run or App Engine targets.
