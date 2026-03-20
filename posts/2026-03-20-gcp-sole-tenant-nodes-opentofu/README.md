# How to Configure GCP Sole-Tenant Nodes with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, Sole-Tenant Nodes, Compliance, OpenTofu, Compute Engine, Security

Description: Learn how to configure GCP Sole-Tenant Nodes with OpenTofu for compliance requirements that mandate physical host isolation from other customers.

## Overview

GCP Sole-Tenant Nodes provide dedicated physical hosts where only your project's VMs run. This is required for BYOL (Bring Your Own License) compliance, regulatory requirements, and workloads that must run on isolated hardware.

## Step 1: Create a Node Template

```hcl
# main.tf - Node template defines the physical node type
resource "google_compute_node_template" "sole_tenant_template" {
  name      = "sole-tenant-node-template"
  region    = "us-central1"

  # Node type determines the physical CPU and memory
  node_type = "n1-node-96-624"  # 96 vCPUs, 624 GB RAM

  # Optional: specify CPU overcommit for better utilization
  cpu_overcommit_type = "ENABLED"

  # Node affinity labels for VM placement
  node_affinity_labels = {
    workload = "compliance-workload"
  }
}
```

## Step 2: Create a Node Group

```hcl
# Node group manages a fleet of sole-tenant nodes
resource "google_compute_node_group" "sole_tenant_group" {
  name        = "sole-tenant-node-group"
  zone        = "us-central1-a"
  description = "Sole-tenant nodes for compliance workloads"

  node_template = google_compute_node_template.sole_tenant_template.id

  # Initial and autoscaling configuration
  initial_size = 2

  autoscaling_policy {
    mode      = "ON"  # "OFF", "ON", or "ONLY_SCALE_OUT"
    min_nodes = 1
    max_nodes = 5
  }

  # Maintenance policy: when can nodes be migrated for maintenance
  maintenance_policy = "RESTART_IN_PLACE"
}
```

## Step 3: Create VMs on Sole-Tenant Nodes

```hcl
# VM that runs exclusively on sole-tenant nodes
resource "google_compute_instance" "sole_tenant_vm" {
  name         = "compliance-workload-vm"
  machine_type = "n1-standard-32"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "windows-cloud/windows-2022"  # Common for BYOL Windows licenses
      size  = 200
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.subnet.self_link
  }

  # Node affinity ensures this VM runs only on sole-tenant nodes
  scheduling {
    on_host_maintenance = "TERMINATE"
    # Node affinity forces placement on the designated node group
    node_affinities {
      key      = "compute.googleapis.com/node-group-name"
      operator = "IN"
      values   = [google_compute_node_group.sole_tenant_group.name]
    }
  }

  # Enable license tracking for BYOL
  enable_display = false
}
```

## Step 4: BYOL License Configuration

```hcl
# Windows BYOL VM on sole-tenant node
resource "google_compute_instance" "byol_windows_vm" {
  name         = "byol-windows-vm"
  machine_type = "n1-standard-16"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      # Use BYOL image for bring-your-own Windows Server license
      image = "windows-cloud/windows-2022-core-for-sole-tenant-byol"
      size  = 100
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.subnet.self_link
  }

  scheduling {
    on_host_maintenance = "TERMINATE"
    node_affinities {
      key      = "compute.googleapis.com/node-group-name"
      operator = "IN"
      values   = [google_compute_node_group.sole_tenant_group.name]
    }
  }
}
```

## Summary

GCP Sole-Tenant Nodes with OpenTofu provide dedicated physical hardware for compliance and BYOL licensing requirements. Node templates define the hardware type, node groups manage capacity and autoscaling, and VM node affinity rules ensure workloads land on the correct isolated hosts.
