# How to Set Up GCP OS Login with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, OS Login, IAM, OpenTofu, Security, SSH Access

Description: Learn how to configure GCP OS Login with OpenTofu to manage SSH access to Compute Engine instances using IAM instead of SSH keys in project metadata.

## Overview

GCP OS Login replaces the traditional SSH key management in project metadata with IAM-based access control. With OS Login, SSH access is governed by IAM roles, providing centralized, auditable access management with MFA support.

## Step 1: Enable OS Login at Project Level

```hcl
# main.tf - Enable OS Login for all VMs in the project
resource "google_compute_project_metadata" "os_login" {
  metadata = {
    enable-oslogin      = "TRUE"
    enable-oslogin-2fa  = "FALSE"  # Set to TRUE to require 2FA
  }
}
```

## Step 2: Create VM with OS Login

```hcl
# VM with OS Login enabled (overrides project metadata if needed)
resource "google_compute_instance" "os_login_vm" {
  name         = "os-login-vm"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.subnet.self_link
  }

  # Instance-level OS Login metadata (can override project metadata)
  metadata = {
    enable-oslogin = "TRUE"
  }

  service_account {
    email  = google_service_account.vm_sa.email
    scopes = ["cloud-platform"]
  }
}
```

## Step 3: Grant IAM Roles for SSH Access

```hcl
# Grant a user OS Login access (non-admin) to a specific VM
resource "google_compute_instance_iam_member" "user_os_login" {
  project       = var.project_id
  zone          = "us-central1-a"
  instance_name = google_compute_instance.os_login_vm.name
  role          = "roles/compute.osLogin"  # Regular user (non-sudo)
  member        = "user:developer@example.com"
}

# Grant admin OS Login access (with sudo) at project level
resource "google_project_iam_member" "admin_os_login" {
  project = var.project_id
  role    = "roles/compute.osAdminLogin"  # Admin access (sudo)
  member  = "user:admin@example.com"
}

# Grant a service account OS Login (for CI/CD pipelines)
resource "google_project_iam_member" "sa_os_login" {
  project = var.project_id
  role    = "roles/compute.osLogin"
  member  = "serviceAccount:${google_service_account.cicd_sa.email}"
}
```

## Step 4: Required IAM for OS Login

```hcl
# Users also need compute.instances.get for instance information
resource "google_project_iam_member" "compute_viewer" {
  project = var.project_id
  role    = "roles/compute.viewer"
  member  = "user:developer@example.com"
}

# Service Account User role required if using service accounts
resource "google_service_account_iam_member" "sa_user" {
  service_account_id = google_service_account.vm_sa.name
  role               = "roles/iam.serviceAccountUser"
  member             = "user:developer@example.com"
}
```

## Step 5: OS Login with 2FA

```hcl
# Enable 2FA for OS Login across the project
resource "google_compute_project_metadata" "os_login_2fa" {
  metadata = {
    enable-oslogin     = "TRUE"
    enable-oslogin-2fa = "TRUE"  # Requires Google 2FA for SSH access
  }
}
```

## Summary

GCP OS Login with OpenTofu replaces static SSH key management with IAM-based access control. Users connect with their Google identity, access is logged in Cloud Audit Logs, and you can enable 2FA for additional security. This is significantly more manageable than distributing SSH keys across teams.
