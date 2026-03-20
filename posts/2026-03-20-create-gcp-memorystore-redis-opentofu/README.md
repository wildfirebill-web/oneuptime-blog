# How to Create GCP Memorystore for Redis with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Memorystore, Redis, Infrastructure as Code

Description: Learn how to create GCP Memorystore for Redis instances with OpenTofu for fully managed in-memory caching on Google Cloud.

GCP Memorystore for Redis is a fully managed Redis service that provides sub-millisecond latency with automatic failover. Managing instances in OpenTofu ensures consistent capacity, network, and security configuration.

## Provider Configuration

```hcl
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
  region  = "us-central1"
}
```

## Basic Redis Instance

```hcl
resource "google_redis_instance" "cache" {
  name           = "myapp-redis"
  tier           = "STANDARD_HA"  # BASIC or STANDARD_HA
  memory_size_gb = 4
  region         = "us-central1"
  redis_version  = "REDIS_7_0"

  # Deploy into a specific VPC
  authorized_network = google_compute_network.main.id
  connect_mode       = "PRIVATE_SERVICE_ACCESS"  # or DIRECT_PEERING

  auth_enabled    = true
  transit_encryption_mode = "SERVER_AUTHENTICATION"

  redis_configs = {
    maxmemory-policy = "allkeys-lru"
    notify-keyspace-events = "Ex"
  }

  maintenance_policy {
    weekly_maintenance_window {
      day = "SUNDAY"
      start_time {
        hours   = 3
        minutes = 0
      }
    }
  }

  labels = {
    environment = "production"
    team        = "backend"
  }
}
```

## Standard HA Instance (Automatic Failover)

```hcl
resource "google_redis_instance" "ha" {
  name           = "myapp-redis-ha"
  tier           = "STANDARD_HA"
  memory_size_gb = 8
  region         = "us-central1"
  redis_version  = "REDIS_7_0"

  # Read replicas for read scaling
  read_replicas_mode  = "READ_REPLICAS_ENABLED"
  replica_count       = 2

  authorized_network = google_compute_network.main.id
  connect_mode       = "PRIVATE_SERVICE_ACCESS"

  auth_enabled            = true
  transit_encryption_mode = "SERVER_AUTHENTICATION"

  redis_configs = {
    maxmemory-policy = "allkeys-lru"
  }
}
```

## Private Service Access Configuration

```hcl
# Required for PRIVATE_SERVICE_ACCESS connect mode
resource "google_compute_global_address" "private_ip_alloc" {
  name          = "memorystore-private-ip"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.main.id
}

resource "google_service_networking_connection" "private_vpc_connection" {
  network                 = google_compute_network.main.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip_alloc.name]
}
```

## Multiple Environments

```hcl
locals {
  redis_config = {
    dev = {
      tier           = "BASIC"
      memory_size_gb = 1
    }
    staging = {
      tier           = "STANDARD_HA"
      memory_size_gb = 2
    }
    production = {
      tier           = "STANDARD_HA"
      memory_size_gb = 8
    }
  }
}

resource "google_redis_instance" "envs" {
  for_each = local.redis_config

  name           = "myapp-redis-${each.key}"
  tier           = each.value.tier
  memory_size_gb = each.value.memory_size_gb
  region         = "us-central1"
  redis_version  = "REDIS_7_0"

  authorized_network = google_compute_network.main.id
  auth_enabled       = true
}
```

## Outputs

```hcl
output "redis_host" {
  value = google_redis_instance.cache.host
}

output "redis_port" {
  value = google_redis_instance.cache.port
}

output "redis_auth_string" {
  value     = google_redis_instance.cache.auth_string
  sensitive = true
}
```

## Conclusion

GCP Memorystore for Redis in OpenTofu provides fully managed, secure Redis with automatic failover. Use STANDARD_HA tier for production workloads to get automatic failover to a replica, enable auth and transit encryption for security, and connect via Private Service Access to keep traffic within your VPC. Monitor memory usage to size instances appropriately.
