# How to Import GCP Compute Instances into OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, GCP, Compute Engine, Import, Google Cloud

Description: Learn how to import existing GCP Compute Engine instances into OpenTofu state, writing matching HCL configurations for instance settings, disks, and network interfaces.

## Introduction

GCP Compute Engine instances created via gcloud, the console, or Deployment Manager can be imported into OpenTofu. The import ID format uses the project, zone, and instance name.

## Step 1: Gather Instance Information

```bash
PROJECT="my-project-id"
ZONE="us-central1-a"
INSTANCE="my-app-vm"

# Get instance details
gcloud compute instances describe $INSTANCE \
  --zone=$ZONE \
  --project=$PROJECT \
  --format=json | jq '{
    machine_type: .machineType | split("/") | last,
    image: .disks[0].source | split("/") | last,
    disk_size: .disks[0].diskSizeGb,
    network: .networkInterfaces[0].network | split("/") | last,
    subnetwork: .networkInterfaces[0].subnetwork | split("/") | last,
    service_account: .serviceAccounts[0].email,
    tags: .tags.items,
    labels: .labels
  }'
```

## Step 2: Write Matching HCL

```hcl
resource "google_compute_instance" "app" {
  name         = "my-app-vm"
  machine_type = "e2-medium"
  zone         = "us-central1-a"
  project      = var.project_id

  # Boot disk configuration
  boot_disk {
    initialize_params {
      image = "ubuntu-os-cloud/ubuntu-2204-lts"
      size  = 30
      type  = "pd-ssd"
    }
  }

  network_interface {
    network    = "my-vpc-network"
    subnetwork = "my-private-subnet"

    # Omit access_config if no external IP
    # access_config {
    #   nat_ip = ""
    # }
  }

  service_account {
    email  = "my-app-sa@my-project-id.iam.gserviceaccount.com"
    scopes = ["cloud-platform"]
  }

  tags = ["app-server", "allow-internal"]

  labels = {
    environment = "prod"
    managed_by  = "opentofu"
  }

  metadata = {
    enable-oslogin = "TRUE"
  }

  lifecycle {
    # Prevent replacement when the boot disk image has a new version
    ignore_changes = [boot_disk[0].initialize_params[0].image]
  }
}
```

## Step 3: Import Block

```hcl
# import.tf
# GCP import ID format: PROJECT/ZONE/INSTANCE_NAME
import {
  to = google_compute_instance.app
  id = "my-project-id/us-central1-a/my-app-vm"
}
```

## Importing Additional Disks

```hcl
resource "google_compute_disk" "data" {
  name    = "my-app-data-disk"
  type    = "pd-ssd"
  zone    = "us-central1-a"
  project = var.project_id
  size    = 100
}

resource "google_compute_attached_disk" "data" {
  disk     = google_compute_disk.data.self_link
  instance = google_compute_instance.app.self_link
  zone     = "us-central1-a"
}

import {
  to = google_compute_disk.data
  id = "my-project-id/us-central1-a/my-app-data-disk"
}

# Attached disk import ID: project/zone/instance/disk
import {
  to = google_compute_attached_disk.data
  id = "my-project-id/us-central1-a/my-app-vm/my-app-data-disk"
}
```

## Handling Preemptible and Spot Instances

```hcl
resource "google_compute_instance" "worker" {
  name         = "batch-worker"
  machine_type = "n2-standard-4"
  zone         = "us-central1-a"

  scheduling {
    preemptible                  = true
    automatic_restart            = false
    on_host_maintenance          = "TERMINATE"
    provisioning_model           = "SPOT"
    instance_termination_action  = "STOP"
  }

  # boot_disk and network_interface as above...
}
```

## Conclusion

GCP Compute instances use the `PROJECT/ZONE/INSTANCE_NAME` format for import IDs. The most important `ignore_changes` setting is the boot disk image reference — GCP image families regularly publish new versions, and without this ignore the instance will appear to need replacement every time the base image is updated.
