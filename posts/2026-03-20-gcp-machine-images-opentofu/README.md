# How to Create GCP Machine Images with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, Machine Image, Compute Engine, OpenTofu, Infrastructure, Golden Image

Description: Learn how to create GCP Machine Images with OpenTofu to capture complete VM state including all disks and configuration for cloning and disaster recovery.

## Overview

GCP Machine Images capture the complete state of a Compute Engine instance, including all attached disks, metadata, and configuration. Unlike snapshots (single disk), machine images can capture multi-disk VMs and serve as golden image sources for new instances.

## Step 1: Create a Machine Image from an Existing VM

```hcl
# main.tf - Source VM to create a machine image from

resource "google_compute_instance" "source_vm" {
  name         = "golden-image-source"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 50
    }
  }

  network_interface {
    network = "default"
  }

  # Setup script to install and configure software
  metadata_startup_script = <<-SCRIPT
    #!/bin/bash
    apt-get update
    apt-get install -y nginx python3 pip
    pip3 install flask gunicorn
    systemctl enable nginx
  SCRIPT
}

# Create a machine image from the configured VM
resource "google_compute_machine_image" "golden_image" {
  name        = "golden-app-image-v1"
  description = "Golden image with nginx and Python app stack"
  source_instance = google_compute_instance.source_vm.self_link

  # Machine image is regional - specifies where image data is stored
  machine_image_encryption_key {
    # Optionally encrypt with a CMEK key
    kms_key_name = google_kms_crypto_key.image_key.id
  }
}
```

## Step 2: Create an Instance from a Machine Image

```hcl
# Create a new VM from the machine image
resource "google_compute_instance" "vm_from_image" {
  name         = "app-server-from-image"
  machine_type = "e2-medium"
  zone         = "us-central1-b"

  # Use the machine image as the source (not a disk image)
  source_machine_image = google_compute_machine_image.golden_image.self_link

  # Network configuration (overrides the source VM's network)
  network_interface {
    subnetwork = google_compute_subnetwork.subnet.self_link
  }

  # Override app settings for the new environment
  metadata = {
    ENVIRONMENT = "production"
  }
}
```

## Step 3: Machine Image with Multiple Disks

```hcl
# Source VM with multiple disks
resource "google_compute_instance" "multi_disk_source" {
  name         = "multi-disk-source"
  machine_type = "n2-standard-4"
  zone         = "us-central1-a"

  # Boot disk
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 50
    }
  }

  # Additional data disk
  attached_disk {
    source      = google_compute_disk.data_disk.self_link
    device_name = "data-disk"
  }

  network_interface {
    network = "default"
  }
}

resource "google_compute_disk" "data_disk" {
  name = "app-data-disk"
  type = "pd-ssd"
  zone = "us-central1-a"
  size = 200
}

# Machine image captures ALL disks (boot + data)
resource "google_compute_machine_image" "complete_image" {
  name            = "complete-app-image"
  source_instance = google_compute_instance.multi_disk_source.self_link
}
```

## Step 4: Outputs

```hcl
output "machine_image_self_link" {
  value       = google_compute_machine_image.golden_image.self_link
  description = "Machine image self link for creating instances"
}
```

## Summary

GCP Machine Images with OpenTofu enable golden image workflows where you configure one VM, capture it as a machine image, and then create multiple identical instances from it. Unlike disk snapshots, machine images capture the entire VM state including all disks, making them ideal for complete environment cloning and DR recovery.
