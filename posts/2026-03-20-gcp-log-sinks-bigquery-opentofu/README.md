# How to Create GCP Log Sinks to BigQuery with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Cloud Logging, BigQuery, Log Analytics, Infrastructure as Code

Description: Learn how to create GCP Cloud Logging sinks that export logs to BigQuery using OpenTofu, enabling SQL-based log analysis, dashboards, and long-term log retention.

GCP Log Sinks route log entries to external destinations for long-term storage and analysis. BigQuery is the most powerful destination for log analysis — it lets you run SQL queries over terabytes of logs with zero operational overhead. Managing log sinks in OpenTofu ensures consistent routing configuration across all your projects.

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
  region  = var.region
}
```

## BigQuery Dataset for Logs

```hcl
resource "google_bigquery_dataset" "logs" {
  dataset_id  = "cloud_logs"
  description = "Centralized log storage for analysis"
  location    = "US"

  default_table_expiration_ms     = 7776000000  # 90 days in milliseconds
  default_partition_expiration_ms = 7776000000

  labels = {
    purpose = "log-analytics"
    managed = "opentofu"
  }

  # Restrict access to authorized users only
  access {
    role          = "OWNER"
    user_by_email = data.google_client_openid_userinfo.me.email
  }

  access {
    role           = "READER"
    group_by_email = var.log_readers_group
  }
}
```

## Project-Level Log Sink to BigQuery

```hcl
# Sink all application logs to BigQuery
resource "google_logging_project_sink" "app_to_bigquery" {
  name    = "app-logs-to-bigquery"
  project = var.project_id

  destination = "bigquery.googleapis.com/projects/${var.project_id}/datasets/${google_bigquery_dataset.logs.dataset_id}"

  # Filter to application logs only (exclude noisy system logs)
  filter = <<-EOT
    resource.type="k8s_container"
    AND resource.labels.cluster_name="${var.cluster_name}"
  EOT

  # Creates a unique service account for this sink
  unique_writer_identity = true

  bigquery_options {
    use_partitioned_tables = true  # Partition by date for cost-efficient queries
  }
}

# Grant the sink's service account write access to BigQuery dataset
resource "google_bigquery_dataset_iam_member" "sink_writer" {
  dataset_id = google_bigquery_dataset.logs.dataset_id
  role       = "roles/bigquery.dataEditor"
  member     = google_logging_project_sink.app_to_bigquery.writer_identity
}
```

## Audit Log Sink to BigQuery

```hcl
resource "google_bigquery_dataset" "audit_logs" {
  dataset_id  = "audit_logs"
  description = "GCP audit logs for compliance and security analysis"
  location    = "US"

  # Keep audit logs for 1 year
  default_table_expiration_ms     = 31536000000
  default_partition_expiration_ms = 31536000000
}

resource "google_logging_project_sink" "audit_to_bigquery" {
  name    = "audit-logs-to-bigquery"
  project = var.project_id

  destination = "bigquery.googleapis.com/projects/${var.project_id}/datasets/${google_bigquery_dataset.audit_logs.dataset_id}"

  filter = <<-EOT
    logName=("projects/${var.project_id}/logs/cloudaudit.googleapis.com%2Factivity"
    OR "projects/${var.project_id}/logs/cloudaudit.googleapis.com%2Fdata_access"
    OR "projects/${var.project_id}/logs/cloudaudit.googleapis.com%2Fsystem_event")
  EOT

  unique_writer_identity = true

  bigquery_options {
    use_partitioned_tables = true
  }
}

resource "google_bigquery_dataset_iam_member" "audit_sink_writer" {
  dataset_id = google_bigquery_dataset.audit_logs.dataset_id
  role       = "roles/bigquery.dataEditor"
  member     = google_logging_project_sink.audit_to_bigquery.writer_identity
}
```

## Organization-Level Sink (Aggregate All Projects)

```hcl
# Aggregate logs from all projects in the organization
resource "google_bigquery_dataset" "org_logs" {
  project     = var.log_project_id  # Centralized logging project
  dataset_id  = "organization_logs"
  description = "Aggregated logs from all GCP projects"
  location    = "US"

  default_partition_expiration_ms = 7776000000  # 90 days
}

resource "google_logging_organization_sink" "org_to_bigquery" {
  name   = "org-logs-to-bigquery"
  org_id = var.org_id

  destination = "bigquery.googleapis.com/projects/${var.log_project_id}/datasets/${google_bigquery_dataset.org_logs.dataset_id}"

  filter = "severity >= WARNING"  # Only WARNING and above to control volume

  include_children = true  # Include logs from all child projects

  unique_writer_identity = true

  bigquery_options {
    use_partitioned_tables = true
  }
}

resource "google_bigquery_dataset_iam_member" "org_sink_writer" {
  project    = var.log_project_id
  dataset_id = google_bigquery_dataset.org_logs.dataset_id
  role       = "roles/bigquery.dataEditor"
  member     = google_logging_organization_sink.org_to_bigquery.writer_identity
}
```

## Saved BigQuery Queries for Log Analysis

```hcl
# Create saved queries for common log analysis tasks
resource "google_bigquery_routine" "top_errors" {
  dataset_id   = google_bigquery_dataset.logs.dataset_id
  routine_id   = "top_errors_last_24h"
  routine_type = "PROCEDURE"
  language     = "SQL"

  definition_body = <<-EOT
    SELECT
      jsonPayload.message AS error_message,
      COUNT(*) AS count,
      MIN(timestamp) AS first_seen,
      MAX(timestamp) AS last_seen
    FROM `${var.project_id}.${google_bigquery_dataset.logs.dataset_id}.*`
    WHERE
      severity = 'ERROR'
      AND timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
    GROUP BY error_message
    ORDER BY count DESC
    LIMIT 20;
  EOT
}
```

## Conclusion

GCP Log Sinks to BigQuery in OpenTofu enable powerful SQL-based log analysis at scale. Use partitioned tables to keep query costs low, create organization-level sinks to aggregate logs from all projects into a central dataset, and use unique_writer_identity so each sink has its own service account with minimal permissions. Combine with BigQuery Data Studio or Looker for live log dashboards, and save common analysis queries as BigQuery procedures for team-wide reuse.
