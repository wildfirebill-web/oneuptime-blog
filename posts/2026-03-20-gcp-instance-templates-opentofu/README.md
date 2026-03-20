# How to Set Up GCP Instance Templates with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, Compute Engine, Instance Templates, OpenTofu, Infrastructure, Autoscaling

Description: Learn how to create GCP Compute Engine instance templates with OpenTofu for use with managed instance groups, autoscaling, and standardized VM configurations.

## Overview

GCP Instance Templates define the configuration for VM instances in Managed Instance Groups (MIGs). They're immutable blueprints that specify machine type, disk, network, and startup scripts. OpenTofu manages template versioning and rollout strategies.

## Step 1: Basic Instance Template

```hcl
# main.tf - Instance template for web server fleet

resource "google_compute_instance_template" "web_template" {
  name_prefix  = "web-server-template-"
  machine_type = "e2-medium"
  region       = "us-central1"

  # Boot disk configuration
  disk {
    source_image = "debian-cloud/debian-12"
    auto_delete  = true
    boot         = true
    disk_size_gb = 50
    disk_type    = "pd-ssd"
  }

  # Network interface
  network_interface {
    subnetwork = google_compute_subnetwork.web_subnet.self_link
  }

  # Startup script runs when the instance boots
  metadata_startup_script = <<-SCRIPT
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    echo "Hello from $(hostname)" > /var/www/html/index.html
    systemctl start nginx
    systemctl enable nginx
  SCRIPT

  service_account {
    email  = google_service_account.vm_sa.email
    scopes = ["cloud-platform"]
  }

  tags = ["web-server", "http-server", "https-server"]

  # Use lifecycle to create before destroying for zero-downtime updates
  lifecycle {
    create_before_destroy = true
  }
}
```

## Step 2: Template with Confidential Computing

```hcl
# Instance template with Shielded VM and Confidential Computing
resource "google_compute_instance_template" "secure_template" {
  name_prefix  = "secure-template-"
  machine_type = "n2d-standard-2"  # AMD required for Confidential VMs
  region       = "us-central1"

  disk {
    source_image = "debian-cloud/debian-12"
    auto_delete  = true
    boot         = true
  }

  network_interface {
    subnetwork = google_compute_subnetwork.web_subnet.self_link
  }

  # Enable Shielded VM features
  shielded_instance_config {
    enable_secure_boot          = true
    enable_vtpm                 = true
    enable_integrity_monitoring = true
  }

  # Enable Confidential Computing
  confidential_instance_config {
    enable_confidential_compute = true
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

## Step 3: Template with Additional Data Disk

```hcl
resource "google_compute_instance_template" "data_template" {
  name_prefix  = "data-server-"
  machine_type = "n2-standard-4"
  region       = "us-central1"

  # Boot disk
  disk {
    source_image = "debian-cloud/debian-12"
    boot         = true
    auto_delete  = true
    disk_size_gb = 50
  }

  # Attached data disk for application data
  disk {
    boot         = false
    auto_delete  = true
    disk_size_gb = 200
    disk_type    = "pd-balanced"
  }

  network_interface {
    subnetwork = google_compute_subnetwork.web_subnet.self_link
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

## Step 4: Outputs

```hcl
output "instance_template_self_link" {
  value       = google_compute_instance_template.web_template.self_link
  description = "Self link to use in managed instance groups"
}
```

## Summary

GCP Instance Templates with OpenTofu provide immutable VM blueprints for managed instance groups. The `create_before_destroy` lifecycle ensures zero-downtime template updates. Use template versioning through `name_prefix` so new templates get unique names, allowing gradual rollout without breaking existing MIGs.
