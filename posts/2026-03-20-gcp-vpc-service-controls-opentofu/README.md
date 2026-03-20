# How to Configure GCP VPC Service Controls with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCP, VPC Service Controls, Security, OpenTofu, Compliance, Data Exfiltration

Description: Learn how to configure GCP VPC Service Controls with OpenTofu to create security perimeters that prevent data exfiltration from sensitive GCP projects.

## Overview

VPC Service Controls create security perimeters around GCP projects and services, preventing data exfiltration even if credentials are compromised. Resources inside the perimeter can communicate with each other but not with resources outside, unless explicitly allowed.

## Step 1: Create an Access Policy

```hcl
# main.tf - Create an access policy (org-level container for perimeters)
resource "google_access_context_manager_access_policy" "policy" {
  parent = "organizations/${var.org_id}"
  title  = "Default Access Policy"
}
```

## Step 2: Create an Access Level

```hcl
# Access level that allows access from specific networks/devices
resource "google_access_context_manager_access_level" "corporate_access" {
  parent = "accessPolicies/${google_access_context_manager_access_policy.policy.name}"
  name   = "accessPolicies/${google_access_context_manager_access_policy.policy.name}/accessLevels/corporate_access"
  title  = "Corporate Network Access"

  basic {
    conditions {
      # Allow access from corporate IP range
      ip_subnetworks = ["203.0.113.0/24", "198.51.100.0/24"]
    }

    conditions {
      # Allow access from managed corporate devices
      device_policy {
        require_screen_lock              = true
        require_corp_owned               = true
        allowed_encryption_statuses      = ["ENCRYPTED"]
      }
    }

    combining_function = "OR"
  }
}
```

## Step 3: Create a Service Perimeter

```hcl
# Service perimeter protecting sensitive projects
resource "google_access_context_manager_service_perimeter" "data_perimeter" {
  parent = "accessPolicies/${google_access_context_manager_access_policy.policy.name}"
  name   = "accessPolicies/${google_access_context_manager_access_policy.policy.name}/servicePerimeters/data_perimeter"
  title  = "Data Science Security Perimeter"

  # Use "PERIMETER_TYPE_REGULAR" for enforcement, "PERIMETER_TYPE_BRIDGE" for communication between perimeters
  perimeter_type = "PERIMETER_TYPE_REGULAR"

  status {
    # Projects included in this perimeter
    resources = [
      "projects/${var.data_project_number}",
    ]

    # GCP services restricted by the perimeter
    restricted_services = [
      "bigquery.googleapis.com",
      "storage.googleapis.com",
      "pubsub.googleapis.com",
    ]

    # Access levels that can cross the perimeter
    access_levels = [
      google_access_context_manager_access_level.corporate_access.name,
    ]

    # Ingress rules - allow specific external identities to access perimeter
    ingress_policies {
      ingress_from {
        identity_type = "SERVICE_ACCOUNT"
        identities    = [
          "serviceAccount:${google_service_account.data_sa.email}",
        ]
      }

      ingress_to {
        resources = ["*"]
        operations {
          service_name = "bigquery.googleapis.com"
          method_selectors {
            method = "google.cloud.bigquery.v2.TableService.ListTables"
          }
        }
      }
    }
  }

  # Use dry_run to test before enforcement
  # lifecycle { prevent_destroy = true }
}
```

## Summary

GCP VPC Service Controls with OpenTofu create an additional layer of security beyond IAM by restricting API access to specific networks, identities, and device states. Perimeters prevent data exfiltration even with stolen credentials, making them essential for compliance-sensitive environments handling PII or regulated data.
