# How to Provision MySQL with Terraform on GCP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Terraform, GCP, Cloud SQL, Infrastructure as Code

Description: Provision a production-ready MySQL Cloud SQL instance on Google Cloud Platform using Terraform with networking, flags, and high availability.

---

## Cloud SQL for MySQL on GCP

Google Cloud SQL provides a fully managed MySQL service on GCP. Using Terraform to provision Cloud SQL instances ensures your database configuration is version-controlled, consistent across environments, and repeatable. The `google` Terraform provider includes comprehensive support for Cloud SQL.

## Provider and Backend Setup

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  backend "gcs" {
    bucket = "my-terraform-state-bucket"
    prefix = "cloud-sql-mysql"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

## Private VPC Networking

Configure private IP access so Cloud SQL is not exposed to the public internet:

```hcl
# network.tf
resource "google_compute_global_address" "private_ip_range" {
  name          = "private-ip-range-mysql"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 24
  network       = var.vpc_self_link
}

resource "google_service_networking_connection" "private_vpc_connection" {
  network                 = var.vpc_self_link
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip_range.name]
}
```

## Cloud SQL MySQL Instance

```hcl
# main.tf
resource "google_sql_database_instance" "mysql" {
  name             = "${var.environment}-mysql-instance"
  database_version = "MYSQL_8_0"
  region           = var.region
  deletion_protection = true

  settings {
    tier              = var.machine_type
    availability_type = var.environment == "production" ? "REGIONAL" : "ZONAL"
    disk_size         = 100
    disk_type         = "PD_SSD"
    disk_autoresize   = true

    ip_configuration {
      ipv4_enabled                                  = false
      private_network                               = var.vpc_self_link
      require_ssl                                   = true
      enable_private_path_for_google_cloud_services = true
    }

    backup_configuration {
      enabled                        = true
      binary_log_enabled             = true
      start_time                     = "02:00"
      location                       = var.region
      transaction_log_retention_days = 7

      backup_retention_settings {
        retained_backups = 7
        retention_unit   = "COUNT"
      }
    }

    maintenance_window {
      day          = 7
      hour         = 4
      update_track = "stable"
    }

    database_flags {
      name  = "slow_query_log"
      value = "on"
    }

    database_flags {
      name  = "long_query_time"
      value = "1"
    }

    database_flags {
      name  = "max_connections"
      value = "500"
    }

    insights_config {
      query_insights_enabled  = true
      query_string_length     = 1024
      record_application_tags = true
      record_client_address   = false
    }
  }

  depends_on = [google_service_networking_connection.private_vpc_connection]
}
```

## Databases and Users

```hcl
# databases.tf
resource "google_sql_database" "app" {
  name      = var.database_name
  instance  = google_sql_database_instance.mysql.name
  charset   = "utf8mb4"
  collation = "utf8mb4_unicode_ci"
}

resource "google_sql_user" "app_user" {
  name     = var.app_username
  instance = google_sql_database_instance.mysql.name
  password = var.app_password
  host     = "%"
}
```

## Outputs

```hcl
# outputs.tf
output "connection_name" {
  description = "Cloud SQL connection name for Cloud SQL Auth Proxy"
  value       = google_sql_database_instance.mysql.connection_name
}

output "private_ip" {
  description = "Private IP address of the Cloud SQL instance"
  value       = google_sql_database_instance.mysql.private_ip_address
}
```

## Deploying

```bash
terraform init
terraform plan -var-file=environments/production.tfvars
terraform apply -var-file=environments/production.tfvars
```

## Summary

Provisioning MySQL on GCP with Terraform involves configuring private VPC networking to avoid public IP exposure, defining a Cloud SQL instance with regional high availability, backup configuration, database flags for tuning and observability, and Query Insights for performance monitoring. The result is a production-ready Cloud SQL instance created consistently through code.
