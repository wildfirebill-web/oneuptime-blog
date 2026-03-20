# How to Deploy Highly Available Applications with OpenTofu on GCP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, High Availability, OpenTofu, Regional MIG, Cloud Load Balancing

Description: Learn how to deploy highly available applications on GCP using OpenTofu with Regional Managed Instance Groups, Global Load Balancing, and health-based autohealing.

## Overview

High availability on GCP uses Regional Managed Instance Groups to distribute VMs across zones, Global HTTP(S) Load Balancing for anycast routing, and autohealing to replace unhealthy instances. OpenTofu provisions the complete HA architecture.

## Step 1: Regional Managed Instance Group

```hcl
# main.tf - Regional MIG for zone-distributed HA

resource "google_compute_instance_template" "app" {
  name_prefix  = "ha-app-"
  machine_type = "n2-standard-4"

  lifecycle {
    create_before_destroy = true
  }

  disk {
    source_image = "projects/debian-cloud/global/images/family/debian-12"
    auto_delete  = true
    boot         = true
    disk_type    = "pd-balanced"
    disk_size_gb = 50
  }

  network_interface {
    subnetwork = google_compute_subnetwork.app.self_link
    # No external IP - access via load balancer
  }

  service_account {
    email  = google_service_account.app.email
    scopes = ["cloud-platform"]
  }

  metadata_startup_script = templatefile("${path.module}/startup.sh", {
    project_id = var.project_id
  })

  tags = ["ha-app", "allow-health-check"]
}

# Regional MIG spans all zones in the region automatically
resource "google_compute_region_instance_group_manager" "app" {
  name   = "ha-app-rmig"
  region = "us-central1"

  base_instance_name = "ha-app"

  version {
    instance_template = google_compute_instance_template.app.id
    name              = "primary"
  }

  target_size = 6  # Distributed evenly across zones (3 zones = 2 per zone)

  named_port {
    name = "http"
    port = 8080
  }

  # Autohealing with health check
  auto_healing_policies {
    health_check      = google_compute_health_check.app.id
    initial_delay_sec = 300
  }

  # Rolling update strategy
  update_policy {
    type                    = "PROACTIVE"
    minimal_action          = "REPLACE"
    max_surge_type          = "FIXED"
    max_surge_fixed         = 3
    max_unavailable_type    = "FIXED"
    max_unavailable_fixed   = 0  # Zero unavailable during updates
    replacement_method      = "SUBSTITUTE"
  }
}
```

## Step 2: Health Check

```hcl
# Health check for autohealing and load balancer
resource "google_compute_health_check" "app" {
  name               = "ha-app-health-check"
  check_interval_sec = 15
  timeout_sec        = 5
  healthy_threshold  = 2
  unhealthy_threshold = 3

  http_health_check {
    port         = 8080
    request_path = "/health/ready"
  }
}
```

## Step 3: Regional Autoscaler

```hcl
# Regional autoscaler for the MIG
resource "google_compute_region_autoscaler" "app" {
  name   = "ha-app-autoscaler"
  region = "us-central1"
  target = google_compute_region_instance_group_manager.app.id

  autoscaling_policy {
    min_replicas    = 3   # Minimum 1 per zone
    max_replicas    = 30
    cooldown_period = 60

    cpu_utilization {
      target = 0.7
    }

    load_balancing_utilization {
      target = 0.8
    }

    scale_in_control {
      max_scaled_in_replicas {
        fixed = 3  # Scale in max 3 at a time
      }
      time_window_sec = 300
    }
  }
}
```

## Step 4: Global Load Balancer Backend

```hcl
resource "google_compute_backend_service" "ha_app" {
  name        = "ha-app-backend"
  protocol    = "HTTP"
  timeout_sec = 30

  # Enable connection draining
  connection_draining_timeout_sec = 30

  backend {
    group           = google_compute_region_instance_group_manager.app.instance_group
    balancing_mode  = "UTILIZATION"
    capacity_scaler = 1.0
  }

  health_checks = [google_compute_health_check.app.id]

  log_config {
    enable      = true
    sample_rate = 0.1  # 10% sampling
  }
}
```

## Summary

Highly available applications on GCP built with OpenTofu use Regional MIGs to automatically distribute instances across all zones in a region. The autohealing policy replaces instances that fail health checks within the `initial_delay_sec` window. The rolling update policy with `max_unavailable_fixed = 0` ensures zero-downtime deployments by requiring all new instances to pass health checks before removing old ones. Global Load Balancing with `connection_draining_timeout_sec` ensures in-flight requests complete before an instance is removed from the backend.
