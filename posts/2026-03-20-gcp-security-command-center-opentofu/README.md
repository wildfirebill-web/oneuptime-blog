# How to Set Up GCP Security Command Center with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, Security Command Center, OpenTofu, Security, Compliance, Threat Detection

Description: Learn how to configure GCP Security Command Center with OpenTofu to enable continuous security monitoring, threat detection, and vulnerability scanning across GCP resources.

## Overview

GCP Security Command Center (SCC) is a risk dashboard and threat intelligence platform for GCP. It discovers misconfigurations, vulnerabilities, and threats across GCP resources. OpenTofu manages notification configurations, security marks, and BigQuery exports.

## Step 1: Enable Security Command Center

```hcl
# main.tf - Enable SCC at organization level

resource "google_security_center_organization_security_health_analytics_custom_module" "scc" {
  organization = var.org_id
}
```

## Step 2: Configure Notification for High-Severity Findings

```hcl
# Create Pub/Sub topic for SCC findings
resource "google_pubsub_topic" "scc_findings" {
  name = "security-command-center-findings"
}

# SCC notification configuration
resource "google_scc_notification_config" "high_severity_alerts" {
  config_id    = "high-severity-findings"
  organization = var.org_id
  description  = "Notify on HIGH and CRITICAL severity findings"
  pubsub_topic = google_pubsub_topic.scc_findings.id

  streaming_config {
    # Filter for critical and high severity findings that are active
    filter = "severity = \"HIGH\" OR severity = \"CRITICAL\" AND state = \"ACTIVE\""
  }
}
```

## Step 3: Enable Security Health Analytics

```hcl
# SCC source for continuous compliance monitoring
resource "google_scc_source" "custom_source" {
  display_name = "Custom Security Scanner"
  organization = var.org_id
  description  = "Custom security findings source for application-level checks"
}
```

## Step 4: Export Findings to BigQuery

```hcl
# BigQuery dataset for SCC findings analysis
resource "google_bigquery_dataset" "scc_exports" {
  dataset_id = "security_command_center_exports"
  location   = "US"
  description = "SCC findings exported for analysis and compliance reporting"
}

# BigQuery export for long-term findings storage and analysis
resource "google_scc_project_notification_config" "bq_export" {
  config_id = "bq-findings-export"
  project   = var.project_id

  pubsub_topic = google_pubsub_topic.scc_findings.id

  streaming_config {
    filter = "state = \"ACTIVE\""
  }
}
```

## Step 5: Add Security Marks

```hcl
# Apply security marks to resources for filtering and grouping
resource "google_scc_source_iam_member" "source_admin" {
  source       = google_scc_source.custom_source.name
  organization = var.org_id
  role         = "roles/securitycenter.findingsEditor"
  member       = "serviceAccount:${google_service_account.security_scanner_sa.email}"
}
```

## Step 6: Outputs

```hcl
output "scc_notification_config" {
  value       = google_scc_notification_config.high_severity_alerts.name
  description = "SCC notification configuration for findings"
}

output "findings_pubsub_topic" {
  value = google_pubsub_topic.scc_findings.id
}
```

## Summary

GCP Security Command Center with OpenTofu provides continuous security posture monitoring across your GCP organization. Notification configurations route high-severity findings to Pub/Sub for real-time alerting, BigQuery exports enable compliance reporting, and Security Health Analytics continuously checks for misconfigurations.
