# How to Set Up GCP Identity-Aware Proxy with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, IAP, Identity-Aware Proxy, OpenTofu, Zero Trust, Security

Description: Learn how to configure GCP Identity-Aware Proxy (IAP) with OpenTofu to secure internal applications with Google identity verification without VPN.

## Overview

GCP Identity-Aware Proxy (IAP) controls access to your applications and VMs based on identity and context rather than network perimeter. It acts as a BeyondCorp access layer, allowing secure access to internal apps without a VPN.

## Step 1: Enable IAP for App Engine or Cloud Run

```hcl
# main.tf - Enable IAP API
resource "google_project_service" "iap" {
  service = "iap.googleapis.com"
}
```

## Step 2: Configure IAP Brand (OAuth Consent Screen)

```hcl
# IAP brand for OAuth consent screen
resource "google_iap_brand" "project_brand" {
  support_email     = "support@example.com"
  application_title = "Internal Engineering Tools"
  project           = var.project_id
}

# IAP OAuth client
resource "google_iap_client" "project_client" {
  display_name = "Internal Tools IAP Client"
  brand        = google_iap_brand.project_brand.name
}
```

## Step 3: Secure a Web App Backend Service

```hcl
# Enable IAP on a backend service (App Engine, Cloud Run, or Load Balancer backend)
resource "google_iap_web_backend_service_iam_binding" "app_iap" {
  project             = var.project_id
  web_backend_service = google_compute_backend_service.app_backend.name
  role                = "roles/iap.httpsResourceAccessor"

  # Allow specific users and groups
  members = [
    "user:engineer@example.com",
    "group:engineering-team@example.com",
    "domain:example.com",  # Allow entire domain
  ]
}
```

## Step 4: Secure a GCE Backend Service

```hcl
# IAP for Compute Engine instances behind a load balancer
resource "google_iap_web_iam_member" "gce_iap" {
  project = var.project_id
  role    = "roles/iap.httpsResourceAccessor"
  member  = "group:developers@example.com"
}
```

## Step 5: Tunnel to VM Instances via IAP

```hcl
# Grant IAP tunnel access to VMs (for SSH/RDP without external IPs)
resource "google_iap_tunnel_instance_iam_member" "vm_tunnel_access" {
  project  = var.project_id
  zone     = "us-central1-a"
  instance = google_compute_instance.private_vm.name
  role     = "roles/iap.tunnelResourceAccessor"
  member   = "user:sysadmin@example.com"
}
```

## Step 6: App Engine IAP

```hcl
# Enable IAP on App Engine
resource "google_iap_web_type_app_engine_iam_binding" "app_engine_iap" {
  project = var.project_id
  app_id  = google_app_engine_application.app.app_id
  role    = "roles/iap.httpsResourceAccessor"

  members = [
    "group:internal-users@example.com",
  ]
}
```

## Summary

GCP Identity-Aware Proxy with OpenTofu replaces VPN-based access with identity-aware, context-sensitive security. By granting IAP roles to users and groups, you control who can access applications and VMs based on Google identity. IAP tunnel access enables SSH/RDP to private VMs without external IPs, fulfilling zero-trust remote access requirements.
