# How to Provision GCP Memorystore Redis with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, GCP, Terraform, Memorystore, Infrastructure as Code

Description: Learn how to provision Google Cloud Memorystore for Redis using Terraform, including VPC configuration, replica count, auth, and maintenance windows.

---

## Prerequisites

- Terraform >= 1.3.0
- Google Cloud SDK installed: `gcloud auth application-default login`
- A GCP project with the Redis API enabled

Enable the API:

```bash
gcloud services enable redis.googleapis.com --project=your-project-id
```

## Provider Configuration

```hcl
# providers.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.3.0"
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

## Basic Memorystore Redis Instance

```hcl
# main.tf

# VPC Network (or use your existing VPC)
resource "google_compute_network" "vpc" {
  name                    = "redis-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "subnet" {
  name          = "redis-subnet"
  ip_cidr_range = "10.0.1.0/24"
  region        = var.region
  network       = google_compute_network.vpc.id
}

# Memorystore Redis Standard Tier
resource "google_redis_instance" "redis" {
  name           = "my-app-redis"
  tier           = "STANDARD_HA"  # BASIC or STANDARD_HA (high availability)
  memory_size_gb = 4
  region         = var.region

  authorized_network = google_compute_network.vpc.id

  # Redis version
  redis_version = "REDIS_7_0"

  # Connect to specific subnet
  connect_mode = "PRIVATE_SERVICE_ACCESS"

  # Auth token for authentication
  auth_enabled = true

  # TLS in transit encryption
  transit_encryption_mode = "SERVER_AUTHENTICATION"  # or "DISABLED"

  # Labels
  labels = {
    environment = var.environment
    managed_by  = "terraform"
  }

  # Maintenance policy
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

  # Redis configuration
  redis_configs = {
    maxmemory-policy = "allkeys-lru"
    notify-keyspace-events = ""
  }
}
```

## High Availability with Read Replicas

Memorystore Standard HA supports up to 5 read replicas:

```hcl
resource "google_redis_instance" "redis_ha" {
  name           = "my-app-redis-ha"
  tier           = "STANDARD_HA"
  memory_size_gb = 8
  region         = var.region

  authorized_network = google_compute_network.vpc.id
  redis_version      = "REDIS_7_0"

  auth_enabled            = true
  transit_encryption_mode = "SERVER_AUTHENTICATION"

  # Read replicas configuration
  replica_count          = 2   # 0-5 replicas
  read_replicas_mode     = "READ_REPLICAS_ENABLED"

  redis_configs = {
    maxmemory-policy    = "allkeys-lru"
    maxmemory-gb        = "6"
    activedefrag        = "yes"
  }

  labels = {
    environment = var.environment
    tier        = "ha"
  }
}
```

## Private Service Access Configuration

For VPC-native Memorystore connectivity:

```hcl
# Reserve IP range for Private Service Access
resource "google_compute_global_address" "private_ip_range" {
  name          = "redis-private-ip-range"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 24
  network       = google_compute_network.vpc.id
}

# Create the peering
resource "google_service_networking_connection" "private_vpc_connection" {
  network                 = google_compute_network.vpc.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip_range.name]
}

# Redis instance using Private Service Access
resource "google_redis_instance" "redis_private" {
  name           = "my-app-redis-private"
  tier           = "STANDARD_HA"
  memory_size_gb = 4
  region         = var.region

  authorized_network = google_compute_network.vpc.id
  connect_mode       = "PRIVATE_SERVICE_ACCESS"

  redis_version = "REDIS_7_0"
  auth_enabled  = true

  depends_on = [google_service_networking_connection.private_vpc_connection]
}
```

## Outputs

```hcl
# outputs.tf
output "redis_host" {
  value       = google_redis_instance.redis.host
  description = "The IP address of the Redis instance"
}

output "redis_port" {
  value       = google_redis_instance.redis.port
  description = "The port of the Redis instance"
}

output "redis_auth_string" {
  value       = google_redis_instance.redis.auth_string
  sensitive   = true
  description = "The AUTH string for the Redis instance"
}

output "redis_read_endpoint" {
  value       = try(google_redis_instance.redis_ha.read_endpoint, "")
  description = "The read endpoint for read replicas"
}

output "redis_read_endpoint_port" {
  value       = try(google_redis_instance.redis_ha.read_endpoint_port, 0)
  description = "The port of the read endpoint"
}
```

## Variables

```hcl
# variables.tf
variable "project_id" {
  description = "GCP project ID"
  type        = string
}

variable "region" {
  description = "GCP region for Redis"
  type        = string
  default     = "us-central1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "prod"
}

variable "redis_memory_gb" {
  description = "Memory size in GB"
  type        = number
  default     = 4
}
```

## Deploying

```bash
# Initialize
terraform init

# Create a terraform.tfvars file
cat > terraform.tfvars << 'EOF'
project_id    = "your-gcp-project-id"
region        = "us-central1"
environment   = "prod"
redis_memory_gb = 4
EOF

# Plan
terraform plan

# Apply
terraform apply

# Get connection details
terraform output redis_host
terraform output -raw redis_auth_string
```

## Connecting to Memorystore Redis

Memorystore Redis is accessible only from within the authorized VPC. Connect from a GCE instance or GKE pod:

```bash
# From a VM in the same VPC
export REDIS_HOST=$(terraform output -raw redis_host)
export REDIS_PORT=$(terraform output redis_port)
export REDIS_AUTH=$(terraform output -raw redis_auth_string)

redis-cli -h $REDIS_HOST -p $REDIS_PORT -a $REDIS_AUTH PING
```

Python connection with TLS:

```python
import redis
import ssl

r = redis.Redis(
    host='10.0.1.3',  # Memorystore private IP
    port=6378,         # TLS port
    password='your-auth-string',
    ssl=True,
    ssl_cert_reqs=ssl.CERT_REQUIRED,
    ssl_ca_certs='/etc/ssl/certs/ca-certificates.crt'
)

r.ping()
```

## Summary

Provisioning GCP Memorystore for Redis with Terraform uses the google_redis_instance resource with tier selection (BASIC or STANDARD_HA), VPC network authorization for private connectivity, and Redis configuration for memory policy and keyspace events. Enable auth_enabled and transit_encryption_mode for security. Use read_replicas_mode with replica_count for read scaling in Standard HA tier. The auth_string output is sensitive and should be stored in Secret Manager rather than passed directly in application configuration.
