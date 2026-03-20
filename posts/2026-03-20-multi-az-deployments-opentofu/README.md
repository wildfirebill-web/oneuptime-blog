# How to Set Up Multi-AZ Deployments with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Multi-AZ, High Availability, OpenTofu, AWS, Azure, GCP

Description: Learn how to configure Multi-AZ deployments using OpenTofu across AWS, Azure, and GCP to protect applications from single availability zone failures.

## Overview

Multi-AZ deployments spread infrastructure across physically separate datacenters within a region. OpenTofu uses data sources to dynamically discover available AZs and distributes resources evenly, avoiding hardcoded zone names.

## Step 1: Dynamic AZ Discovery

```hcl
# main.tf - Dynamically discover AZs for portability

data "aws_availability_zones" "available" {
  state = "available"
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}

locals {
  # Use first 3 AZs dynamically
  azs = slice(data.aws_availability_zones.available.names, 0, 3)

  # Calculate subnet CIDRs dynamically for each AZ
  private_cidrs = [for i, az in local.azs : cidrsubnet("10.0.0.0/16", 8, i)]
  public_cidrs  = [for i, az in local.azs : cidrsubnet("10.0.0.0/16", 8, i + 10)]
}
```

## Step 2: AWS Multi-AZ RDS

```hcl
# RDS with Multi-AZ enabled
resource "aws_db_instance" "multi_az" {
  identifier     = "app-db-multi-az"
  engine         = "postgres"
  instance_class = "db.r6g.large"
  multi_az       = true  # Automatic standby in another AZ

  # AWS manages the standby instance and automatic failover
  # Typical failover time: 60-120 seconds
}

# ElastiCache Redis with Multi-AZ
resource "aws_elasticache_replication_group" "redis_multi_az" {
  replication_group_id = "app-redis-multi-az"
  description          = "Redis with multi-AZ"
  node_type            = "cache.r6g.large"
  num_cache_clusters   = 3  # One per AZ

  automatic_failover_enabled  = true
  multi_az_enabled            = true
  preferred_cache_cluster_azs = local.azs
}
```

## Step 3: Azure Zone-Redundant Services

```hcl
# Zone-redundant Azure SQL (Business Critical supports zone redundancy)
resource "azurerm_mssql_database" "zone_redundant" {
  name         = "app-db-zone-redundant"
  server_id    = azurerm_mssql_server.sql.id
  sku_name     = "BusinessCritical_Gen5_4"  # Supports zone redundancy

  zone_redundant = true  # Replicate across AZs
}

# Zone-redundant Azure Cache for Redis
resource "azurerm_redis_cache" "zone_redundant" {
  name                = "app-redis-zone-redundant"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  capacity            = 1
  family              = "P"  # Premium supports zone redundancy
  sku_name            = "Premium"
  zones               = ["1", "2", "3"]
}
```

## Step 4: GCP Multi-Zone Instances

```hcl
# GCP Regional persistent disk (replicated across zones)
resource "google_compute_region_disk" "app_data" {
  name   = "app-data-regional"
  type   = "pd-ssd"
  region = "us-central1"

  replica_zones = [
    "us-central1-a",
    "us-central1-b"
  ]

  size = 100
}

# Spread instances across zones using for_each
resource "google_compute_instance" "app" {
  for_each = toset(["us-central1-a", "us-central1-b", "us-central1-c"])

  name         = "app-${replace(each.value, "us-central1-", "")}"
  machine_type = "n2-standard-4"
  zone         = each.value

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      type  = "pd-balanced"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.app.self_link
  }
}
```

## Summary

Multi-AZ deployments configured with OpenTofu use dynamic AZ discovery via data sources to avoid hardcoded zone names that break when AZs change. AWS RDS Multi-AZ maintains a synchronous standby replica for 60-120 second automatic failover. Azure zone-redundant services synchronously replicate within the region for near-instant failover. GCP Regional MIGs and disks provide zone-level redundancy managed by Google's infrastructure automatically.
