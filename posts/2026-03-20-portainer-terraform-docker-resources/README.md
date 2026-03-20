# How to Manage Docker Resources via Portainer Terraform Provider (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Terraform, Docker, Infrastructure, DevOps

Description: Learn how to manage Docker containers, networks, volumes, and images through the Portainer Terraform provider for fully declarative container infrastructure management.

## Introduction

The Portainer Terraform provider allows you to manage not just Portainer configuration but also the underlying Docker resources through Portainer's API proxy. This means you can define containers, networks, and volumes as Terraform resources and manage their entire lifecycle declaratively.

## Prerequisites

- Portainer Terraform provider configured
- Docker environment registered in Portainer
- Terraform v1.0+

## Step 1: Managing Docker Containers

```hcl
# containers.tf

# Simple container deployment

resource "portainer_container" "nginx" {
  endpoint_id = portainer_environment.production.id
  name        = "nginx-reverse-proxy"
  image       = "nginx:1.25"

  restart_policy = "unless-stopped"

  ports = [
    {
      host_port      = 80
      container_port = 80
      protocol       = "tcp"
    },
    {
      host_port      = 443
      container_port = 443
      protocol       = "tcp"
    }
  ]

  volumes = [
    {
      host_path      = "/opt/nginx/conf.d"
      container_path = "/etc/nginx/conf.d"
      read_only      = false
    }
  ]

  environment = {
    NGINX_HOST = "example.com"
    NGINX_PORT = "80"
  }

  labels = {
    "app"         = "nginx"
    "managed-by"  = "terraform"
    "environment" = "production"
  }
}

# Container with resource limits
resource "portainer_container" "app" {
  endpoint_id = portainer_environment.production.id
  name        = "my-application"
  image       = "ghcr.io/my-org/myapp:${var.app_image_tag}"

  restart_policy = "unless-stopped"

  resource_limits = {
    cpu_shares      = 512     # Relative CPU weight
    memory          = 536870912  # 512MB in bytes
    memory_swap     = 1073741824 # 1GB in bytes
  }

  ports = [
    {
      host_ip        = "127.0.0.1"  # Bind to localhost only
      host_port      = 3000
      container_port = 3000
      protocol       = "tcp"
    }
  ]

  environment = {
    NODE_ENV     = "production"
    DATABASE_URL = var.database_url
    REDIS_URL    = "redis://redis:6379"
  }

  network_mode = "app-network"
}
```

## Step 2: Managing Docker Networks

```hcl
# networks.tf

# Custom bridge network
resource "portainer_network" "app_network" {
  endpoint_id = portainer_environment.production.id
  name        = "app-network"
  driver      = "bridge"

  ipam_config = {
    subnet  = "172.30.0.0/16"
    gateway = "172.30.0.1"
  }

  options = {
    "com.docker.network.bridge.name" = "app-br"
  }

  labels = {
    "managed-by" = "terraform"
    "project"    = "myapp"
  }
}

# Overlay network for Swarm
resource "portainer_network" "swarm_overlay" {
  endpoint_id = portainer_environment.production_swarm.id
  name        = "swarm-overlay"
  driver      = "overlay"
  attachable  = true  # Allow standalone containers to attach

  ipam_config = {
    subnet = "10.20.0.0/16"
  }
}
```

## Step 3: Managing Docker Volumes

```hcl
# volumes.tf

# Named volume for persistent data
resource "portainer_volume" "postgres_data" {
  endpoint_id = portainer_environment.production.id
  name        = "postgres_data"
  driver      = "local"

  labels = {
    "managed-by" = "terraform"
    "backup"     = "required"
    "app"        = "postgresql"
  }
}

resource "portainer_volume" "redis_data" {
  endpoint_id = portainer_environment.production.id
  name        = "redis_data"
  driver      = "local"

  labels = {
    "managed-by" = "terraform"
    "app"        = "redis"
  }
}

# NFS volume
resource "portainer_volume" "shared_uploads" {
  endpoint_id = portainer_environment.production.id
  name        = "shared_uploads"
  driver      = "local"

  driver_opts = {
    type   = "nfs"
    o      = "addr=192.168.1.100,rw"
    device = ":/mnt/shared/uploads"
  }
}
```

## Step 4: Complete Application Stack with All Resources

```hcl
# complete_app.tf - Full application with all Docker resources

# Network
resource "portainer_network" "myapp_net" {
  endpoint_id = portainer_environment.production.id
  name        = "myapp-network"
  driver      = "bridge"
}

# Volumes
resource "portainer_volume" "db_data" {
  endpoint_id = portainer_environment.production.id
  name        = "myapp-db-data"
  driver      = "local"
}

resource "portainer_volume" "app_uploads" {
  endpoint_id = portainer_environment.production.id
  name        = "myapp-uploads"
  driver      = "local"
}

# Database container
resource "portainer_container" "database" {
  endpoint_id = portainer_environment.production.id
  name        = "myapp-db"
  image       = "postgres:15-alpine"

  restart_policy = "unless-stopped"

  environment = {
    POSTGRES_DB       = "myapp"
    POSTGRES_USER     = "myapp"
    POSTGRES_PASSWORD = var.db_password
  }

  volumes = [
    {
      volume_name    = portainer_volume.db_data.name
      container_path = "/var/lib/postgresql/data"
    }
  ]

  network_mode = portainer_network.myapp_net.name
}

# Application container
resource "portainer_container" "app" {
  endpoint_id = portainer_environment.production.id
  name        = "myapp-web"
  image       = "ghcr.io/myorg/myapp:${var.image_tag}"

  restart_policy = "unless-stopped"

  ports = [
    {
      host_port      = 8080
      container_port = 3000
    }
  ]

  environment = {
    DATABASE_URL = "postgresql://myapp:${var.db_password}@myapp-db:5432/myapp"
  }

  volumes = [
    {
      volume_name    = portainer_volume.app_uploads.name
      container_path = "/app/uploads"
    }
  ]

  network_mode = portainer_network.myapp_net.name

  depends_on = [portainer_container.database]
}

# Outputs
output "app_url" {
  value = "http://${var.host_ip}:8080"
}
```

## Step 5: Validate and Apply

```bash
# Validate configuration
terraform validate

# Plan - see all resources to be created
terraform plan

# Apply - create all Docker resources
terraform apply

# View created resources in Portainer
# Or check via Docker
docker ps
docker network ls
docker volume ls
```

## Conclusion

Managing Docker resources via the Portainer Terraform provider gives you full declarative control over containers, networks, and volumes. Resources are defined in code, changes are reviewed through pull requests, and the Terraform state file tracks all managed resources. This approach is particularly valuable for reproducible deployment environments, disaster recovery scenarios, and ensuring environment parity between staging and production.
