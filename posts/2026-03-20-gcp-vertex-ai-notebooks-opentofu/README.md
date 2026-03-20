# How to Create GCP Vertex AI Notebooks with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Vertex AI, Notebooks, Machine Learning, Infrastructure as Code

Description: Learn how to create GCP Vertex AI Workbench managed notebooks and user-managed notebook instances for ML development using OpenTofu.

## Introduction

Vertex AI Workbench provides Jupyter-based notebook environments for ML development on GCP. Managed notebooks are fully hosted by Google; user-managed notebooks give more control over the VM. OpenTofu manages both types as code.

## Enabling APIs

```hcl
resource "google_project_service" "notebooks" {
  project = var.project_id
  service = "notebooks.googleapis.com"
}

resource "google_project_service" "aiplatform" {
  project = var.project_id
  service = "aiplatform.googleapis.com"
}
```

## Managed Notebook Instance (Workbench)

```hcl
resource "google_notebooks_instance" "managed_nb" {
  name     = "${var.app_name}-notebook-${var.environment}"
  project  = var.project_id
  location = "${var.region}-a"  # notebooks are zonal resources

  machine_type = "n1-standard-4"

  vm_image {
    project      = "deeplearning-platform-release"
    image_family = "tf-latest-cpu"  # TensorFlow CPU image
    # or "tf-latest-gpu" for GPU-enabled
  }

  # Attach a GPU (if using a GPU VM type)
  # accelerator_config {
  #   type       = "NVIDIA_TESLA_T4"
  #   core_count = 1
  # }

  install_gpu_driver = false

  service_account = google_service_account.notebooks.email

  # Network configuration
  network = google_compute_network.main.id
  subnet  = google_compute_subnetwork.private.id

  no_public_ip    = true  # use private IP only
  no_proxy_access = false  # allow JupyterLab proxy access

  labels = {
    environment = var.environment
    managed_by  = "opentofu"
  }

  metadata = {
    notebook-disable-root = "true"
    framework             = "tensorflow"
  }
}
```

## Service Account for Notebooks

```hcl
resource "google_service_account" "notebooks" {
  account_id   = "${var.app_name}-notebooks-sa"
  display_name = "Vertex AI Notebooks SA"
  project      = var.project_id
}

resource "google_project_iam_member" "notebooks_storage" {
  project = var.project_id
  role    = "roles/storage.objectAdmin"
  member  = "serviceAccount:${google_service_account.notebooks.email}"
}

resource "google_project_iam_member" "notebooks_aiplatform" {
  project = var.project_id
  role    = "roles/aiplatform.user"
  member  = "serviceAccount:${google_service_account.notebooks.email}"
}
```

## Vertex AI Workbench User-Managed Notebook (Legacy)

```hcl
resource "google_notebooks_instance" "user_managed" {
  name     = "${var.app_name}-user-nb-${var.environment}"
  project  = var.project_id
  location = "${var.region}-b"

  machine_type = "e2-standard-4"

  container_image {
    repository = "gcr.io/deeplearning-platform-release/base-cpu"
    tag        = "latest"
  }

  service_account  = google_service_account.notebooks.email
  network          = google_compute_network.main.id
  subnet           = google_compute_subnetwork.private.id
  no_public_ip     = true
  no_proxy_access  = false

  post_startup_script = "gs://${var.setup_bucket}/scripts/notebook_setup.sh"
}
```

## Outputs

```hcl
output "notebook_proxy_uri" {
  description = "JupyterLab access URL"
  value       = google_notebooks_instance.managed_nb.proxy_uri
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

Vertex AI Workbench notebooks provide managed Jupyter environments optimized for ML workflows on GCP. OpenTofu manages notebook instances, service accounts with appropriate permissions, and post-startup scripts — creating a consistent, reproducible data science environment.
