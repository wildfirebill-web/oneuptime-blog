# How to Create AKS Clusters with Custom Node Pools Using OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, AKS, Node Pools, Kubernetes, Autoscaling, Infrastructure as Code

Description: Learn how to create AKS clusters with custom node pools using OpenTofu, including system and user node pools with different VM sizes, autoscaling, and node pool taints for workload isolation.

## Introduction

AKS node pools are groups of VMs with the same configuration within a Kubernetes cluster. Every AKS cluster has at least one system node pool for Kubernetes system components (kube-system), and you can add user node pools for application workloads. Different node pools can use different VM sizes, OS types, availability zones, and autoscaling settings—enabling you to match compute resources to specific workload requirements (GPU nodes for ML, high-memory for databases, general-purpose for APIs).

## Prerequisites

- OpenTofu v1.6+
- Azure credentials with AKS permissions
- A Resource Group and Virtual Network with subnets

## Step 1: Create AKS Cluster with System Node Pool

```hcl
resource "azurerm_kubernetes_cluster" "main" {
  name                = "${var.project_name}-aks"
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = var.project_name
  kubernetes_version  = "1.28"

  # System node pool (required)
  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D4s_v3"
    node_count          = 2
    min_count           = 2
    max_count           = 5
    enable_auto_scaling = true

    vnet_subnet_id = var.system_subnet_id
    zones          = ["1", "2", "3"]

    # Taint system pool to prevent user workloads
    only_critical_addons_enabled = true

    os_disk_size_gb      = 128
    os_disk_type         = "Ephemeral"  # Faster than managed disk for node OS
    max_pods             = 30

    upgrade_settings {
      max_surge = "33%"
    }

    node_labels = {
      "nodepool-type" = "system"
    }
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    network_policy    = "calico"
    load_balancer_sku = "standard"
  }

  tags = {
    Name        = "${var.project_name}-aks"
    Environment = var.environment
  }
}
```

## Step 2: Add User Node Pools

```hcl
# General-purpose workload pool
resource "azurerm_kubernetes_cluster_node_pool" "app" {
  name                  = "app"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D4s_v3"
  node_count            = 3
  min_count             = 2
  max_count             = 20
  enable_auto_scaling   = true

  vnet_subnet_id = var.app_subnet_id
  zones          = ["1", "2", "3"]

  os_disk_type = "Ephemeral"
  max_pods     = 30

  node_labels = {
    "workload-type" = "general"
  }

  upgrade_settings {
    max_surge = "33%"
  }
}

# GPU node pool for ML workloads
resource "azurerm_kubernetes_cluster_node_pool" "gpu" {
  name                  = "gpu"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_NC6s_v3"
  node_count            = 0  # Start with 0, scale when needed
  min_count             = 0
  max_count             = 5
  enable_auto_scaling   = true

  vnet_subnet_id = var.gpu_subnet_id
  zones          = ["1"]  # GPUs may not be in all zones

  # Node labels for scheduling
  node_labels = {
    "workload-type" = "gpu"
    "gpu-type"      = "nvidia-tesla-v100"
  }

  # Taint to prevent non-GPU workloads from scheduling here
  node_taints = ["nvidia.com/gpu=true:NoSchedule"]

  os_disk_size_gb = 256
  max_pods        = 10
}

# High-memory pool for cache/data workloads
resource "azurerm_kubernetes_cluster_node_pool" "memory" {
  name                  = "memory"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_E8s_v3"  # Memory-optimized
  node_count            = 2
  min_count             = 1
  max_count             = 10
  enable_auto_scaling   = true

  vnet_subnet_id = var.app_subnet_id
  zones          = ["1", "2", "3"]

  node_labels = {
    "workload-type" = "memory-intensive"
  }

  node_taints = ["memory-optimized=true:PreferNoSchedule"]
}
```

## Step 3: Node Pool Upgrade Settings

```hcl
# Blue pool (current version)
resource "azurerm_kubernetes_cluster_node_pool" "blue" {
  name                  = "blue"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D4s_v3"
  node_count            = 5
  kubernetes_version    = "1.28"  # Pin to specific version for blue/green upgrade

  upgrade_settings {
    max_surge = "1"  # Add 1 node at a time during upgrades
  }

  node_labels = {
    "pool-color" = "blue"
  }
}
```

## Step 4: Deploy

```bash
tofu init
tofu plan
tofu apply

# Get cluster credentials
az aks get-credentials \
  --resource-group <rg> \
  --name <cluster-name>

# List node pools
kubectl get nodes -L agentpool,workload-type

# Scale a specific node pool
az aks nodepool scale \
  --resource-group <rg> \
  --cluster-name <cluster-name> \
  --name gpu \
  --node-count 2
```

## Conclusion

Use `only_critical_addons_enabled = true` on system node pools to prevent user workloads from running alongside Kubernetes system components, reducing risk of resource starvation on critical cluster services. Use `os_disk_type = "Ephemeral"` for node pools when possible—ephemeral OS disks use VM host cache and provide better read performance, lower latency, and zero OS disk cost. Set GPU node pools to `min_count = 0` with Cluster Autoscaler to scale down to zero during idle periods, dramatically reducing GPU costs.
