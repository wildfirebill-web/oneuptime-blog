# How to Configure GKE Logging and Monitoring with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GKE, Logging, Monitoring, OpenTofu, Observability, GCP

Description: Learn how to configure GKE cluster logging and monitoring with OpenTofu including Cloud Logging, Cloud Monitoring, and managed Prometheus integration.

## Overview

GKE integrates with Cloud Logging and Cloud Monitoring to provide comprehensive observability. OpenTofu configures logging service, monitoring service, managed Prometheus, and log-based metrics for complete cluster observability.

## Step 1: Configure Logging and Monitoring on Cluster

```hcl
# main.tf - GKE cluster with logging and monitoring

resource "google_container_cluster" "monitored_cluster" {
  name     = "monitored-gke-cluster"
  location = "us-central1"

  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name

  # Logging service
  logging_service = "logging.googleapis.com/kubernetes"

  # Monitoring service
  monitoring_service = "monitoring.googleapis.com/kubernetes"

  # Configure what to log
  logging_config {
    enable_components = [
      "SYSTEM_COMPONENTS",  # kube-system, apiserver, scheduler, etc.
      "WORKLOADS",          # Application container logs
    ]
  }

  # Configure what to monitor
  monitoring_config {
    enable_components = [
      "SYSTEM_COMPONENTS",
      "APISERVER",
      "SCHEDULER",
      "CONTROLLER_MANAGER",
      "STORAGE",
      "HPA",
      "POD",
      "DAEMONSET",
      "DEPLOYMENT",
      "STATEFULSET",
    ]

    # Enable Managed Prometheus for metrics collection
    managed_prometheus {
      enabled = true
    }
  }
}
```

## Step 2: Log-Based Metrics for Alerting

```hcl
# Create a log-based metric for error rates
resource "google_logging_metric" "error_rate_metric" {
  name        = "gke-application-errors"
  description = "Count of application error logs in GKE"
  project     = var.project_id

  filter = <<-FILTER
    resource.type="k8s_container"
    resource.labels.cluster_name="${google_container_cluster.monitored_cluster.name}"
    severity>=ERROR
  FILTER

  metric_descriptor {
    metric_kind = "DELTA"
    value_type  = "INT64"
    unit        = "1"
    display_name = "GKE Application Errors"
  }
}
```

## Step 3: Alerting Policies

```hcl
# Alert when application error count exceeds threshold
resource "google_monitoring_alert_policy" "high_error_rate" {
  display_name = "GKE High Application Error Rate"
  combiner     = "OR"

  conditions {
    display_name = "Error rate too high"

    condition_threshold {
      filter     = "metric.type=\"logging.googleapis.com/user/gke-application-errors\" resource.type=\"k8s_container\""
      duration   = "300s"
      comparison = "COMPARISON_GT"
      threshold_value = 50

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_RATE"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.name]
}

resource "google_monitoring_notification_channel" "email" {
  display_name = "Ops Email Alert"
  type         = "email"

  labels = {
    email_address = "ops@example.com"
  }
}
```

## Step 4: Dashboard for GKE Cluster

```hcl
resource "google_monitoring_dashboard" "gke_dashboard" {
  dashboard_json = jsonencode({
    displayName = "GKE Cluster Overview"
    gridLayout = {
      columns = "2"
      widgets = [
        {
          title = "Pod Count"
          xyChart = {
            dataSets = [{
              timeSeriesQuery = {
                timeSeriesFilter = {
                  filter = "metric.type=\"kubernetes.io/container/cpu/request_cores\" resource.type=\"k8s_container\""
                }
              }
            }]
          }
        }
      ]
    }
  })
}
```

## Summary

GKE logging and monitoring configured with OpenTofu provides comprehensive observability from the start. Managed Prometheus handles workload metrics collection, Cloud Logging captures container logs, and log-based metrics bridge the gap between logs and alerting. This approach eliminates the need to self-manage Prometheus or logging agents.
