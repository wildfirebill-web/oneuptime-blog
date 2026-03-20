# How to Manage Portainer Environments with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Terraform, Environments, Infrastructure as Code, DevOps

Description: Learn how to create, configure, and manage Portainer environments (endpoints) using the Terraform provider.

## Managing Environments as Code

Environments in Portainer represent your Docker and Kubernetes targets. Managing them with Terraform means new environments are registered programmatically when infrastructure is provisioned, and removed when decommissioned.

## Docker Standalone Environment

```hcl
# docker-environments.tf

# Local Docker socket environment

resource "portainer_environment" "local" {
  name = "local-docker"
  type = 1  # Docker standalone
  url  = "unix:///var/run/docker.sock"
}

# Remote Docker host via TLS
resource "portainer_environment" "remote_docker" {
  name = "prod-docker-host"
  type = 1
  url  = "tcp://192.168.1.100:2376"

  # TLS configuration for secure connection
  tls                = true
  tls_ca_cert_file   = file("${path.module}/certs/ca.pem")
  tls_cert_file      = file("${path.module}/certs/cert.pem")
  tls_key_file       = file("${path.module}/certs/key.pem")
}
```

## Docker Swarm Environment (via Agent)

```hcl
# swarm-environment.tf
resource "portainer_environment" "swarm" {
  name = "production-swarm"
  type = 2  # Docker standalone via Portainer agent

  url = "tcp://swarm-manager.mycompany.com:9001"

  # Assign to a group for organization
  group_id = portainer_environment_group.production.id

  # Apply tags
  tag_ids = [
    portainer_tag.production.id,
    portainer_tag.swarm.id
  ]

  # Security settings
  allow_bind_mounts    = false
  allow_privileged     = false
}
```

## Kubernetes Environment

```hcl
# kubernetes-environment.tf

# Kubernetes via kubeconfig
resource "portainer_environment" "k8s_prod" {
  name = "k8s-production"
  type = 5  # Kubernetes via kubeconfig

  kubernetes = {
    kubeconfig = file("${path.module}/kubeconfig/prod-kubeconfig.yaml")
  }
}

# Kubernetes via Portainer agent
resource "portainer_environment" "k8s_staging" {
  name = "k8s-staging"
  type = 7  # Kubernetes via agent

  url = "tcp://k8s-agent.staging.mycompany.com:9001"

  kubernetes_configuration = {
    use_load_balancer    = true
    use_server_metrics   = true
    storage_classes      = ["standard", "ssd"]
    restrict_default_ns  = true
  }
}
```

## Environment Groups and Tags

```hcl
# organization.tf
resource "portainer_environment_group" "production" {
  name = "Production"
}

resource "portainer_environment_group" "staging" {
  name = "Staging"
}

resource "portainer_tag" "production" {
  name = "production"
}

resource "portainer_tag" "swarm" {
  name = "swarm"
}
```

## Provisioning Environments Alongside Infrastructure

Combine with other Terraform providers to register environments as VMs are created:

```hcl
# Provision a VM and register it in Portainer
resource "hcloud_server" "docker_host" {
  name        = "docker-prod-01"
  server_type = "cx31"
  image       = "ubuntu-22.04"
  location    = "nbg1"

  user_data = file("scripts/install-docker-and-agent.sh")
}

# Register the new server as a Portainer environment
resource "portainer_environment" "docker_prod_01" {
  name = hcloud_server.docker_host.name
  type = 2
  url  = "tcp://${hcloud_server.docker_host.ipv4_address}:9001"

  depends_on = [hcloud_server.docker_host]
}
```

## Removing Environments

```bash
# Remove an environment from Portainer (doesn't affect the server/cluster)
terraform destroy -target=portainer_environment.staging_old
```

## Outputs for Downstream Use

```hcl
output "environment_ids" {
  description = "Map of environment names to their Portainer IDs"
  value = {
    local    = portainer_environment.local.id
    swarm    = portainer_environment.swarm.id
    k8s_prod = portainer_environment.k8s_prod.id
  }
}
```

## Conclusion

Managing Portainer environments with Terraform closes the loop between infrastructure provisioning and container platform configuration. When you provision a new server with Terraform, you can simultaneously register it as a Portainer environment in the same plan/apply cycle.
