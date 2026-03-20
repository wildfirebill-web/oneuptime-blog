# How to Manage Docker Resources via Portainer Terraform Provider

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Terraform, Docker, Infrastructure as Code, Automation

Description: Learn how to manage Docker containers, networks, and volumes in Portainer environments using the Terraform provider.

## Overview

Beyond stacks and environments, the Portainer Terraform provider supports managing individual Docker resources — giving you fine-grained control over standalone containers, custom networks, and named volumes as code.

## Managing Docker Networks

```hcl
# networks.tf

# Create a custom bridge network
resource "portainer_docker_network" "app_network" {
  endpoint_id = portainer_environment.production.id

  name   = "app-network"
  driver = "bridge"

  ipam_config = [
    {
      subnet  = "172.20.0.0/16"
      gateway = "172.20.0.1"
    }
  ]

  options = {
    "com.docker.network.bridge.name" = "app-br0"
  }
}

# Create an overlay network for Swarm
resource "portainer_docker_network" "swarm_overlay" {
  endpoint_id = portainer_environment.swarm.id

  name       = "swarm-overlay"
  driver     = "overlay"
  attachable = true

  driver_options = {
    "com.docker.network.driver.mtu" = "1450"
  }
}
```

## Managing Docker Volumes

```hcl
# volumes.tf

resource "portainer_docker_volume" "postgres_data" {
  endpoint_id = portainer_environment.production.id

  name   = "postgres-data"
  driver = "local"

  driver_options = {
    type   = "none"
    device = "/mnt/fast-disk/postgres"
    o      = "bind"
  }

  labels = {
    "managed-by" = "terraform"
    "app"        = "database"
  }
}

resource "portainer_docker_volume" "uploads" {
  endpoint_id = portainer_environment.production.id
  name        = "app-uploads"
  driver      = "local"
}
```

## Managing Standalone Containers

```hcl
# containers.tf

resource "portainer_container" "redis" {
  endpoint_id = portainer_environment.production.id
  name        = "redis"

  image           = "redis:7-alpine"
  restart_policy  = "unless-stopped"

  # Port bindings
  port_bindings = [
    {
      host_port      = "6379"
      container_port = "6379"
      protocol       = "tcp"
    }
  ]

  # Volume mounts
  volumes = [
    {
      volume_name    = portainer_docker_volume.redis_data.name
      container_path = "/data"
    }
  ]

  # Environment variables
  env = [
    "REDIS_PASSWORD=${var.redis_password}"
  ]

  # Network configuration
  network_mode = "bridge"
  networks = [portainer_docker_network.app_network.name]

  # Resource limits
  memory_limit = 512  # MB
  cpu_limit    = 0.5
}
```

## Complete Application Stack with Individual Resources

```hcl
# main.tf - Deploy app components as individual Terraform resources

# Step 1: Create the network
resource "portainer_docker_network" "myapp" {
  endpoint_id = portainer_environment.production.id
  name        = "myapp-net"
  driver      = "bridge"
}

# Step 2: Create the data volume
resource "portainer_docker_volume" "db_data" {
  endpoint_id = portainer_environment.production.id
  name        = "myapp-db-data"
}

# Step 3: Deploy database container
resource "portainer_container" "postgres" {
  endpoint_id = portainer_environment.production.id
  name        = "myapp-postgres"
  image       = "postgres:15-alpine"

  env = [
    "POSTGRES_DB=myapp",
    "POSTGRES_PASSWORD=${var.db_password}"
  ]

  volumes = [{
    volume_name    = portainer_docker_volume.db_data.name
    container_path = "/var/lib/postgresql/data"
  }]

  networks       = [portainer_docker_network.myapp.name]
  restart_policy = "unless-stopped"
}

# Step 4: Deploy the application container
resource "portainer_container" "app" {
  endpoint_id = portainer_environment.production.id
  name        = "myapp-web"
  image       = "registry.mycompany.com/myapp:${var.image_tag}"

  env = [
    "DB_HOST=myapp-postgres",
    "DB_PASSWORD=${var.db_password}"
  ]

  port_bindings = [{ host_port = "8080", container_port = "8080" }]
  networks      = [portainer_docker_network.myapp.name]
  restart_policy = "unless-stopped"

  depends_on = [portainer_container.postgres]
}
```

## Conclusion

The Portainer Terraform provider's Docker resource management capabilities let you define your entire containerized infrastructure as code. For complex multi-container applications, consider using `portainer_stack` (Compose-based) for simpler management, and individual container resources for standalone services with specific configurations.
