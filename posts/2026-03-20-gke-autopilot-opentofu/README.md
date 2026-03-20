# How to Deploy GKE Autopilot with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, GKE, Kubernetes, Autopilot, Terraform, Infrastructure as Code

Description: Learn how to deploy a Google Kubernetes Engine Autopilot cluster with OpenTofu, including network configuration, node pool settings, workload identity, and kubectl access.

---

GKE Autopilot is a fully managed Kubernetes mode where Google manages the control plane, node pools, and infrastructure. You pay only for what your pods use. This guide covers deploying GKE Autopilot with OpenTofu.

---

## Prerequisites

```hcl
# providers.tf
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

variable "project_id" {}
variable "region" { default = "us-central1" }
```

---

## Create a VPC for GKE

```hcl
# network.tf
resource "google_compute_network" "gke_vpc" {
  name                    = "gke-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "gke_subnet" {
  name          = "gke-subnet"
  ip_cidr_range = "10.0.0.0/20"
  network       = google_compute_network.gke_vpc.id
  region        = var.region

  # Secondary ranges for pods and services
  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.64.0.0/16"
  }

  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.128.0.0/20"
  }
}
```

---

## Create GKE Autopilot Cluster

```hcl
# cluster.tf
resource "google_container_cluster" "autopilot" {
  name     = "autopilot-cluster"
  location = var.region

  # Enable Autopilot mode
  enable_autopilot = true

  # Network configuration
  network    = google_compute_network.gke_vpc.id
  subnetwork = google_compute_subnetwork.gke_subnet.id

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  # Private cluster (recommended for production)
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "0.0.0.0/0"
      display_name = "all"
    }
  }

  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Release channel
  release_channel {
    channel = "REGULAR"
  }

  # Logging and monitoring
  logging_service    = "logging.googleapis.com/kubernetes"
  monitoring_service = "monitoring.googleapis.com/kubernetes"

  # Maintenance window
  maintenance_policy {
    recurring_window {
      start_time = "2026-01-01T04:00:00Z"
      end_time   = "2026-01-01T08:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA,SU"
    }
  }

  resource_labels = {
    environment = "production"
    managed_by  = "opentofu"
  }
}
```

---

## Workload Identity for GKE Pods

```hcl
# workload_identity.tf
resource "google_service_account" "app" {
  account_id   = "gke-app-sa"
  display_name = "GKE App Service Account"
}

# Grant the SA access to needed services
resource "google_project_iam_member" "app_storage" {
  project = var.project_id
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.app.email}"
}

# Bind the GCP SA to the Kubernetes SA
resource "google_service_account_iam_binding" "workload_identity" {
  service_account_id = google_service_account.app.name
  role               = "roles/iam.workloadIdentityUser"

  members = [
    "serviceAccount:${var.project_id}.svc.id.goog[production/app-sa]"
  ]
}
```

---

## Cloud NAT for Private Cluster Egress

```hcl
resource "google_compute_router" "gke_router" {
  name    = "gke-router"
  network = google_compute_network.gke_vpc.id
  region  = var.region
}

resource "google_compute_router_nat" "gke_nat" {
  name                               = "gke-nat"
  router                             = google_compute_router.gke_router.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}
```

---

## Outputs

```hcl
# outputs.tf
output "cluster_name" {
  value = google_container_cluster.autopilot.name
}

output "cluster_endpoint" {
  value     = google_container_cluster.autopilot.endpoint
  sensitive = true
}

output "cluster_ca_certificate" {
  value     = google_container_cluster.autopilot.master_auth.0.cluster_ca_certificate
  sensitive = true
}
```

---

## Configure kubectl Access

```bash
# After tofu apply
gcloud container clusters get-credentials autopilot-cluster \
    --region us-central1 \
    --project my-project

# Verify
kubectl get nodes
# In Autopilot, nodes appear when pods are scheduled
# kubectl get pods --all-namespaces
```

---

## Deploy a Sample Workload

```yaml
# sample-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
```

---

## Best Practices

1. **Use private clusters** for production — enable_private_nodes = true
2. **Configure Cloud NAT** for private clusters to pull images from the internet
3. **Use Workload Identity** instead of node service accounts for pod GCP access
4. **Set resource requests** on all pods — Autopilot requires them for scheduling
5. **Use REGULAR release channel** — it receives updates after they've been validated

---

## Conclusion

GKE Autopilot with OpenTofu gives you a fully managed Kubernetes cluster with minimal configuration. Enable Autopilot mode, configure VPC and secondary ranges, set up Workload Identity, and deploy — Google handles everything else.

---

*Monitor your GKE Autopilot workloads with [OneUptime](https://oneuptime.com) — Kubernetes-native observability.*
