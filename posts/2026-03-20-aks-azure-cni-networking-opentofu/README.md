# How to Configure AKS with Azure CNI Networking Using OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, AKS, Azure CNI, Kubernetes Networking, VNet Integration, Infrastructure as Code

Description: Learn how to configure AKS with Azure CNI networking using OpenTofu to assign VNet IP addresses directly to pods for native Azure network integration and Network Policy enforcement.

## Introduction

Azure CNI (Container Network Interface) assigns real Azure VNet IP addresses to every pod, making pods first-class citizens in the VNet. This enables direct pod-to-pod communication across VNets (peering), Network Policy enforcement using Azure or Calico, and access to Azure PaaS services via Service Endpoints or Private Endpoints from pods. Unlike kubenet (which uses NAT), Azure CNI requires more IP address planning since every pod consumes a VNet IP.

## Prerequisites

- OpenTofu v1.6+
- Azure credentials with AKS and Network permissions
- A VNet with subnets large enough for nodes + pods

## Step 1: Plan IP Addresses

Azure CNI reserves IPs per node: `max_pods` per node × number of nodes must fit in the subnet CIDR. For 10 nodes × 30 pods = 300 pod IPs + node IPs → use at least a /23 (512 IPs) subnet.

```hcl
# AKS subnet sized for 30 nodes × 30 pods = 900 IPs + 30 nodes

resource "azurerm_subnet" "aks" {
  name                 = "aks-subnet"
  resource_group_name  = var.resource_group_name
  virtual_network_name = var.vnet_name
  address_prefixes     = ["10.1.0.0/21"]  # /21 = 2048 IPs

  service_endpoints = ["Microsoft.Storage", "Microsoft.Sql"]
}
```

## Step 2: Create AKS Cluster with Azure CNI

```hcl
resource "azurerm_kubernetes_cluster" "azure_cni" {
  name                = "${var.project_name}-aks"
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = var.project_name
  kubernetes_version  = "1.28"

  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D4s_v3"
    node_count          = 3
    min_count           = 3
    max_count           = 10
    enable_auto_scaling = true

    vnet_subnet_id = azurerm_subnet.aks.id
    max_pods       = 30  # IPs per node reserved in subnet

    os_disk_type = "Ephemeral"
    zones        = ["1", "2", "3"]
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"      # Azure CNI
    network_policy    = "azure"      # Azure Network Policy or "calico"
    load_balancer_sku = "standard"
    service_cidr      = "10.200.0.0/16"   # Must not overlap with VNet
    dns_service_ip    = "10.200.0.10"     # Within service_cidr
  }

  tags = {
    Name = "${var.project_name}-aks-azure-cni"
  }
}

# Grant AKS cluster network contributor on the VNet
resource "azurerm_role_assignment" "aks_network" {
  scope                = var.vnet_id
  role_definition_name = "Network Contributor"
  principal_id         = azurerm_kubernetes_cluster.azure_cni.identity[0].principal_id
}
```

## Step 3: Azure CNI Overlay (Reduces IP Consumption)

Azure CNI Overlay uses a private CIDR for pods (not VNet IPs), combining the management simplicity of Azure CNI with reduced IP address consumption.

```hcl
resource "azurerm_kubernetes_cluster" "cni_overlay" {
  name                = "${var.project_name}-aks-overlay"
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = "${var.project_name}-overlay"
  kubernetes_version  = "1.28"

  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D4s_v3"
    node_count          = 3
    enable_auto_scaling = true
    min_count           = 3
    max_count           = 20

    vnet_subnet_id = azurerm_subnet.aks.id
    max_pods       = 250  # Higher pod density with overlay
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin      = "azure"
    network_plugin_mode = "overlay"  # Azure CNI Overlay
    network_policy      = "calico"
    pod_cidr            = "192.168.0.0/16"  # Private CIDR for pods (not VNet)
    service_cidr        = "10.200.0.0/16"
    dns_service_ip      = "10.200.0.10"
  }
}
```

## Step 4: Network Policy Enforcement

```yaml
# kubernetes/network-policy.yaml (apply after cluster creation)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# Get credentials
az aks get-credentials \
  --resource-group <rg> \
  --name <cluster-name>

# Verify pod networking (pods should have VNet IPs)
kubectl get pods -A -o wide

# Check node IP allocation
az network vnet subnet show \
  --resource-group <rg> \
  --vnet-name <vnet-name> \
  --name aks-subnet \
  --query "ipConfigurations[].privateIPAddress"
```

## Conclusion

Choose Azure CNI for workloads that require Network Policy enforcement, direct pod access from on-premises, or pod-level access to Service Endpoints-use CNI Overlay mode to avoid the IP exhaustion problem. Calculate subnet size as: `(max_pods × max_nodes) + max_nodes + headroom`; always add at least 30% headroom for upgrades (which require additional nodes). With standard Azure CNI, set `max_pods` between 30-50 per node to balance pod density with subnet IP consumption; with CNI Overlay, you can use up to 250 pods per node since pods use a separate CIDR.
