# How to Create DigitalOcean Kubernetes Clusters with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, DigitalOcean, Kubernetes, DOKS, Infrastructure as Code

Description: Learn how to create and configure DigitalOcean Kubernetes (DOKS) clusters with OpenTofu, including node pools and kubeconfig retrieval.

DigitalOcean Kubernetes Service (DOKS) provides managed Kubernetes clusters. With OpenTofu, you can define clusters, node pools, and retrieve kubeconfig credentials as code, enabling reproducible Kubernetes environments.

## Provider Configuration

```hcl
terraform {
  required_providers {
    digitalocean = {
      source  = "digitalocean/digitalocean"
      version = "~> 2.0"
    }
  }
}

provider "digitalocean" {
  token = var.do_token
}
```

## Creating a Kubernetes Cluster

```hcl
resource "digitalocean_kubernetes_cluster" "main" {
  name    = "production-cluster"
  region  = "nyc3"
  version = "1.32.2-do.0"  # Use: doctl kubernetes options versions

  # Default node pool is required
  node_pool {
    name       = "worker-pool"
    size       = "s-2vcpu-4gb"
    node_count = 3

    labels = {
      pool = "worker"
      env  = "production"
    }

    tags = ["worker", "production"]
  }

  tags = ["production", "opentofu"]
}
```

## Adding Extra Node Pools

```hcl
# Add a high-memory node pool for memory-intensive workloads
resource "digitalocean_kubernetes_node_pool" "highmem" {
  cluster_id = digitalocean_kubernetes_cluster.main.id

  name       = "highmem-pool"
  size       = "m-4vcpu-32gb"
  node_count = 2

  labels = {
    pool     = "highmem"
    workload = "memory-intensive"
  }

  taint {
    key    = "workload"
    value  = "highmem"
    effect = "NoSchedule"
  }
}
```

## Auto-Scaling Node Pools

```hcl
resource "digitalocean_kubernetes_node_pool" "autoscaling" {
  cluster_id = digitalocean_kubernetes_cluster.main.id

  name = "autoscaling-pool"
  size = "s-2vcpu-4gb"

  # Enable auto-scaling instead of fixed node_count
  auto_scale = true
  min_nodes  = 2
  max_nodes  = 10
}
```

## Retrieving the Kubeconfig

```hcl
# Output the kubeconfig for use with kubectl
output "kubeconfig" {
  value     = digitalocean_kubernetes_cluster.main.kube_config[0].raw_config
  sensitive = true
}

# Save kubeconfig to a file using local_file
resource "local_file" "kubeconfig" {
  content         = digitalocean_kubernetes_cluster.main.kube_config[0].raw_config
  filename        = "${path.module}/kubeconfig.yaml"
  file_permission = "0600"
}
```

```bash
# After apply, use the kubeconfig
export KUBECONFIG=./kubeconfig.yaml
kubectl get nodes
```

## Placing the Cluster in a VPC

```hcl
resource "digitalocean_vpc" "k8s" {
  name     = "k8s-vpc"
  region   = "nyc3"
  ip_range = "10.20.0.0/16"
}

resource "digitalocean_kubernetes_cluster" "private" {
  name     = "private-cluster"
  region   = "nyc3"
  version  = "1.32.2-do.0"
  vpc_uuid = digitalocean_vpc.k8s.id

  node_pool {
    name       = "default"
    size       = "s-2vcpu-4gb"
    node_count = 3
  }
}
```

## Waiting for the Cluster to Be Ready

OpenTofu waits for the cluster to reach `running` state automatically. Use `timeouts` for clusters that take longer to provision:

```hcl
resource "digitalocean_kubernetes_cluster" "main" {
  # ...
  timeouts {
    create = "30m"
  }
}
```

## Conclusion

DigitalOcean Kubernetes clusters are straightforward to provision with OpenTofu. Define the cluster with at least one node pool, optionally add specialized pools with taints for workload isolation, enable auto-scaling for production workloads, and retrieve the kubeconfig as a sensitive output for use in subsequent pipeline steps.
