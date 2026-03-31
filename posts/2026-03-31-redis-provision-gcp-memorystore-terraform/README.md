# How to Provision GCP Memorystore Redis with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Terraform, GCP, Memorystore, Infrastructure As Code, Cloud

Description: Provision Google Cloud Memorystore for Redis using Terraform with the google provider, configuring tier, auth, VPC peering, and read replicas for production workloads.

---

## Prerequisites

- Terraform >= 1.5
- Google Cloud SDK authenticated (`gcloud auth application-default login`)
- `google` Terraform provider
- A GCP project with the `redis.googleapis.com` API enabled

## Enable Required APIs

```bash
gcloud services enable redis.googleapis.com
gcloud services enable compute.googleapis.com
```

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
  region  = var.region
}

variable "project_id" {
  type = string
}

variable "region" {
  type    = string
  default = "us-central1"
}
```

## VPC Network for Private Access

```hcl
resource "google_compute_network" "vpc" {
  name                    = "redis-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "subnet" {
  name          = "redis-subnet"
  ip_cidr_range = "10.10.0.0/24"
  region        = var.region
  network       = google_compute_network.vpc.id
}

resource "google_compute_global_address" "private_ip" {
  name          = "google-managed-services-range"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.vpc.id
}

resource "google_service_networking_connection" "private_vpc_connection" {
  network                 = google_compute_network.vpc.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip.name]
}
```

## Memorystore Redis Instance - Standard Tier

```hcl
resource "google_redis_instance" "redis" {
  name           = "myapp-redis"
  tier           = "STANDARD_HA"
  memory_size_gb = 4
  region         = var.region

  location_id             = "${var.region}-a"
  alternative_location_id = "${var.region}-b"

  redis_version     = "REDIS_7_0"
  display_name      = "MyApp Redis"
  reserved_ip_range = "10.10.1.0/29"

  auth_enabled            = true
  transit_encryption_mode = "SERVER_AUTHENTICATION"

  authorized_network = google_compute_network.vpc.id

  redis_configs = {
    maxmemory-policy = "allkeys-lru"
    notify-keyspace-events = ""
  }

  maintenance_policy {
    weekly_maintenance_window {
      day = "SUNDAY"
      start_time {
        hours   = 2
        minutes = 0
        seconds = 0
        nanos   = 0
      }
    }
  }

  depends_on = [google_service_networking_connection.private_vpc_connection]
}
```

## Basic Tier (Cost-Optimized, No HA)

```hcl
resource "google_redis_instance" "redis_basic" {
  name           = "myapp-redis-dev"
  tier           = "BASIC"
  memory_size_gb = 1
  region         = var.region
  redis_version  = "REDIS_7_0"
  auth_enabled   = true
  authorized_network = google_compute_network.vpc.id
}
```

## Read Replicas (STANDARD_HA + read replicas)

```hcl
resource "google_redis_instance" "redis_with_replicas" {
  name           = "myapp-redis-prod"
  tier           = "STANDARD_HA"
  memory_size_gb = 8
  region         = var.region
  redis_version  = "REDIS_7_0"
  auth_enabled   = true
  authorized_network = google_compute_network.vpc.id

  replica_count         = 2
  read_replicas_mode    = "READ_REPLICAS_ENABLED"
}
```

## Outputs

```hcl
output "redis_host" {
  value = google_redis_instance.redis.host
}

output "redis_port" {
  value = google_redis_instance.redis.port
}

output "redis_auth_string" {
  value     = google_redis_instance.redis.auth_string
  sensitive = true
}

output "redis_server_ca_cert" {
  value     = google_redis_instance.redis.server_ca_certs[0].cert
  sensitive = true
}
```

## Deploy

```bash
terraform init
terraform plan -var="project_id=my-gcp-project"
terraform apply -var="project_id=my-gcp-project"
```

Get connection details:

```bash
terraform output -raw redis_host
terraform output -raw redis_auth_string
```

## Connect to Memorystore Redis

Since Memorystore is VPC-native, connect from a VM or GKE workload in the same VPC:

```python
import redis
import ssl

REDIS_HOST = "10.10.1.3"
REDIS_PORT = 6378
REDIS_AUTH = "your-auth-string"

r = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    password=REDIS_AUTH,
    ssl=True,
    ssl_cert_reqs=ssl.CERT_REQUIRED,
    ssl_ca_certs="/path/to/server_ca.pem",
    decode_responses=True
)
r.ping()
```

## Import Existing Instance

```bash
terraform import google_redis_instance.redis projects/my-project/locations/us-central1/instances/myapp-redis
```

## Summary

Terraform provisions GCP Memorystore Redis using the `google_redis_instance` resource with STANDARD_HA tier for production workloads. VPC peering ensures private network access without public exposure, and AUTH plus TLS provide security in transit. Read replicas improve read throughput for read-heavy workloads, and the maintenance policy schedules automatic patching during off-peak hours.
