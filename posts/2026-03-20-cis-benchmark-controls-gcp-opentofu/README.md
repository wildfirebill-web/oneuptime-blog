# How to Implement CIS Benchmark Controls with OpenTofu on GCP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, CIS Benchmark, GCP Security, Compliance, Infrastructure as Code

Description: Learn how to implement CIS Google Cloud Platform Foundations Benchmark controls with OpenTofu to secure your GCP projects.

The CIS GCP Foundations Benchmark provides security recommendations for GCP. OpenTofu lets you enforce these controls via organization policies, IAM bindings, logging sinks, and resource configurations.

## Section 1: IAM

```hcl
# CIS 1.1 — Avoid using corporate login credentials (service accounts for automation)
# Enforced via organization policy

# CIS 1.4 — Ensure that service account keys are rotated within 90 days
# This is enforced via an organization policy constraint
resource "google_organization_policy" "disable_sa_key_creation" {
  org_id     = data.google_organization.main.org_id
  constraint = "constraints/iam.disableServiceAccountKeyCreation"

  boolean_policy {
    enforced = true  # Prevent creation of service account keys
  }
}

# CIS 1.5 — Ensure that service accounts do not have admin privileges
# Enforced via IAM binding reviews (detected via policy-as-code tools)
```

## Section 2: Logging and Monitoring

```hcl
# CIS 2.1 — Ensure that Cloud Audit Logs are configured for all services
resource "google_project_iam_audit_config" "all_services" {
  project = var.project_id
  service = "allServices"

  audit_log_config {
    log_type = "DATA_READ"
  }
  audit_log_config {
    log_type = "DATA_WRITE"
  }
  audit_log_config {
    log_type = "ADMIN_READ"
  }
}

# CIS 2.2 — Ensure that sinks are configured for all log entries
resource "google_logging_project_sink" "all_logs" {
  name        = "all-logs-sink"
  project     = var.project_id
  destination = "storage.googleapis.com/${google_storage_bucket.audit_logs.name}"
  filter      = ""  # Empty = all logs

  unique_writer_identity = true
}

# CIS 2.11 — Ensure that the log metric filter and alerts exist for project ownership changes
resource "google_logging_metric" "project_ownership" {
  name   = "project-ownership-changes"
  project = var.project_id
  filter = "protoPayload.serviceName=\"cloudresourcemanager.googleapis.com\" AND ProjectOwnership OR projectOwnerInvitee"

  metric_descriptor {
    metric_kind = "DELTA"
    value_type  = "INT64"
  }
}

resource "google_monitoring_alert_policy" "project_ownership" {
  display_name = "Project Ownership Changes"
  project      = var.project_id

  conditions {
    display_name = "Project ownership change detected"
    condition_threshold {
      filter          = "metric.type=\"logging.googleapis.com/user/project-ownership-changes\""
      comparison      = "COMPARISON_GT"
      threshold_value = 0
      duration        = "0s"
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.name]
}
```

## Section 3: Networking

```hcl
# CIS 3.1 — Ensure the default network does not exist in projects
resource "google_compute_project_metadata" "disable_default_network" {
  # This is enforced via organization policy
}

resource "google_organization_policy" "skip_default_network" {
  org_id     = data.google_organization.main.org_id
  constraint = "constraints/compute.skipDefaultNetworkCreation"

  boolean_policy { enforced = true }
}

# CIS 3.6 — Ensure SSH access from the internet is blocked
resource "google_compute_firewall" "deny_ssh_internet" {
  name    = "deny-ssh-from-internet"
  network = google_compute_network.main.id

  deny {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["0.0.0.0/0"]
  priority      = 1000
}
```

## Section 4: Virtual Machines

```hcl
# CIS 4.1 — Ensure that instances are not configured to use the default service account
# with full API access
resource "google_compute_instance" "cis_compliant" {
  name         = "compliant-instance"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  service_account {
    # Use a dedicated service account, not the Compute default SA
    email  = google_service_account.app.email
    scopes = ["cloud-platform"]  # Use cloud-platform with fine-grained IAM
  }

  # CIS 4.4 — Ensure OS login is enabled
  metadata = {
    enable-oslogin = "TRUE"
  }

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
    # CIS 4.7 — Ensure VM disks are encrypted with CMEK
    kms_key_self_link = google_kms_crypto_key.vm_disk.id
  }
}
```

## Conclusion

CIS GCP Benchmark controls are implemented through a combination of organization policies (for project-wide enforcement), IAM audit config (for comprehensive logging), logging sinks (for log export), monitoring metrics and alerts (for real-time detection), and resource-level settings (OS Login, CMEK, no default network). Use organization-level constraints to enforce controls across all projects automatically.
