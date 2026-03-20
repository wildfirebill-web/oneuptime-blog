# How to Configure GCP Private Google Access with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, Private Google Access, Networking, OpenTofu, Security, VPC

Description: Learn how to configure GCP Private Google Access with OpenTofu to allow VMs without external IPs to access Google APIs and services using internal IPs only.

## Overview

Private Google Access lets VMs without external IP addresses connect to Google APIs and services (like Cloud Storage, BigQuery, Pub/Sub) using Google's private network. This improves security by keeping API traffic off the public internet.

## Step 1: Enable Private Google Access on Subnet

```hcl
# main.tf - Subnet with Private Google Access enabled
resource "google_compute_subnetwork" "private_subnet" {
  name          = "private-subnet"
  network       = google_compute_network.vpc.self_link
  region        = "us-central1"
  ip_cidr_range = "10.0.1.0/24"

  # Enable Private Google Access - allows VMs without external IPs
  # to reach Google APIs via internal routes
  private_ip_google_access = true

  private_ipv6_google_access = "DISABLE_GOOGLE_ACCESS"  # IPv6 option
}
```

## Step 2: Create VM Without External IP

```hcl
# VM with no external IP that accesses Google APIs via Private Google Access
resource "google_compute_instance" "private_vm" {
  name         = "private-api-vm"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.private_subnet.self_link
    # No access_config block = no external IP assigned
  }

  service_account {
    email  = google_service_account.api_sa.email
    scopes = ["cloud-platform"]
  }

  metadata_startup_script = <<-SCRIPT
    #!/bin/bash
    # This VM can call Google APIs without an external IP
    # because Private Google Access is enabled on the subnet
    gcloud storage ls gs://my-bucket/  # Works without external IP
  SCRIPT
}
```

## Step 3: Add DNS and Routes for Private Google Access

```hcl
# Add routes to Google API IP ranges for Private Google Access
# These routes are typically auto-created but can be explicit
resource "google_compute_route" "private_google_api_route" {
  name             = "private-google-api-route"
  network          = google_compute_network.vpc.name
  dest_range       = "199.36.153.8/30"  # restricted.googleapis.com
  priority         = 1000
  next_hop_gateway = "default-internet-gateway"
}

# Private DNS zone for googleapis.com
resource "google_dns_managed_zone" "googleapis_private" {
  name       = "googleapis-private-zone"
  dns_name   = "googleapis.com."
  visibility = "private"

  private_visibility_config {
    networks {
      network_url = google_compute_network.vpc.id
    }
  }
}

# CNAME record pointing to restricted.googleapis.com
resource "google_dns_record_set" "googleapis_cname" {
  name         = "*.googleapis.com."
  managed_zone = google_dns_managed_zone.googleapis_private.name
  type         = "CNAME"
  ttl          = 300
  rrdatas      = ["restricted.googleapis.com."]
}
```

## Step 4: Restricted vs. Private API Access

```hcl
# Route to restricted.googleapis.com (supports VPC Service Controls)
resource "google_compute_route" "restricted_api_route" {
  name             = "restricted-api-route"
  network          = google_compute_network.vpc.name
  dest_range       = "199.36.153.4/30"  # private.googleapis.com
  priority         = 1000
  next_hop_gateway = "default-internet-gateway"
}
```

## Summary

GCP Private Google Access with OpenTofu allows VMs without public IPs to access Google services securely. Enable it on subnets to keep API traffic on Google's private network. For VPC Service Controls compliance, use `restricted.googleapis.com` instead of `private.googleapis.com` to ensure API access respects service perimeter boundaries.
