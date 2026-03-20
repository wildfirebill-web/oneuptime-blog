# How to Set Up AKS Private Clusters with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, AKS, Private Cluster, Private Endpoint, Network Security, Infrastructure as Code

Description: Learn how to create AKS private clusters with OpenTofu to keep the Kubernetes API server endpoint completely private within your VNet, eliminating public API server exposure.

## Introduction

AKS private clusters configure the Kubernetes API server with a Private Endpoint inside your VNet, making it accessible only from within the VNet or connected networks (peered VNets, VPN, ExpressRoute). This eliminates the attack surface of a publicly-accessible API server and is required for high-security environments with strict network isolation requirements. kubectl commands and CI/CD pipelines must run from within the private network or through a jump box/Bastion.

## Prerequisites

- OpenTofu v1.6+
- Azure credentials with AKS permissions
- A VNet with Private DNS Zone support (or use Azure Private DNS Zone)

## Step 1: Create AKS Private Cluster

```hcl
resource "azurerm_kubernetes_cluster" "private" {
  name                = "${var.project_name}-aks-private"
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = "${var.project_name}-private"
  kubernetes_version  = "1.28"

  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D4s_v3"
    node_count          = 3
    min_count           = 3
    max_count           = 10
    enable_auto_scaling = true
    vnet_subnet_id      = var.aks_subnet_id
    zones               = ["1", "2", "3"]

    upgrade_settings {
      max_surge = "33%"
    }
  }

  identity {
    type = "SystemAssigned"
  }

  # Make API server private
  private_cluster_enabled             = true
  private_dns_zone_id                 = "System"  # Use Azure-managed private DNS
  private_cluster_public_fqdn_enabled = false      # Disable public FQDN

  # Optional: custom private DNS zone (for more control)
  # private_dns_zone_id = azurerm_private_dns_zone.aks.id

  network_profile {
    network_plugin    = "azure"
    network_policy    = "calico"
    load_balancer_sku = "standard"
    service_cidr      = "10.200.0.0/16"
    dns_service_ip    = "10.200.0.10"
  }

  oms_agent {
    log_analytics_workspace_id = var.log_analytics_workspace_id
  }

  tags = {
    Name        = "${var.project_name}-aks-private"
    Environment = var.environment
  }
}
```

## Step 2: Custom Private DNS Zone

```hcl
# Create custom private DNS zone for the API server
resource "azurerm_private_dns_zone" "aks" {
  name                = "privatelink.${var.location}.azmk8s.io"
  resource_group_name = var.resource_group_name
}

# Link to AKS VNet
resource "azurerm_private_dns_zone_virtual_network_link" "aks" {
  name                  = "aks-dns-link"
  resource_group_name   = var.resource_group_name
  private_dns_zone_name = azurerm_private_dns_zone.aks.name
  virtual_network_id    = var.vnet_id
  registration_enabled  = false
}

# Link to admin/management VNet for kubectl access
resource "azurerm_private_dns_zone_virtual_network_link" "admin" {
  name                  = "admin-dns-link"
  resource_group_name   = var.resource_group_name
  private_dns_zone_name = azurerm_private_dns_zone.aks.name
  virtual_network_id    = var.admin_vnet_id
  registration_enabled  = false
}

resource "azurerm_kubernetes_cluster" "private_custom_dns" {
  name                = "${var.project_name}-aks-private"
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = "${var.project_name}-private"

  default_node_pool {
    name           = "system"
    vm_size        = "Standard_D4s_v3"
    node_count     = 3
    vnet_subnet_id = var.aks_subnet_id
  }

  identity {
    type = "SystemAssigned"
  }

  private_cluster_enabled = true
  private_dns_zone_id     = azurerm_private_dns_zone.aks.id  # Custom zone

  network_profile {
    network_plugin    = "azure"
    load_balancer_sku = "standard"
  }
}

# Grant AKS identity DNS Zone Contributor to manage DNS records
resource "azurerm_role_assignment" "aks_dns" {
  scope                = azurerm_private_dns_zone.aks.id
  role_definition_name = "Private DNS Zone Contributor"
  principal_id         = azurerm_kubernetes_cluster.private_custom_dns.identity[0].principal_id
}
```

## Step 3: Jump Box for Cluster Access

```hcl
# Admin VM in the same VNet for kubectl access
resource "azurerm_linux_virtual_machine" "jumpbox" {
  name                = "${var.project_name}-jumpbox"
  resource_group_name = var.resource_group_name
  location            = var.location
  size                = "Standard_B2s"

  network_interface_ids           = [azurerm_network_interface.jumpbox.id]
  admin_username                  = "azureuser"
  disable_password_authentication = true

  admin_ssh_key {
    username   = "azureuser"
    public_key = var.ssh_public_key
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  # Install kubectl and az cli via cloud-init
  custom_data = base64encode(<<-EOT
    #!/bin/bash
    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl
    curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg
    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
    apt-get update && apt-get install -y kubectl
    curl -sL https://aka.ms/InstallAzureCLIDeb | bash
  EOT
  )

  identity {
    type = "SystemAssigned"
  }
}
```

## Step 4: CI/CD with Private Cluster

```hcl
# Self-hosted runner subnet for GitHub Actions/Azure DevOps
resource "azurerm_subnet" "cicd_runner" {
  name                 = "cicd-runner-subnet"
  resource_group_name  = var.resource_group_name
  virtual_network_name = var.vnet_name
  address_prefixes     = ["10.0.10.0/24"]
}

# Azure Container Instance for self-hosted CI/CD runner
resource "azurerm_container_group" "gh_runner" {
  name                = "${var.project_name}-gh-runner"
  location            = var.location
  resource_group_name = var.resource_group_name
  ip_address_type     = "Private"
  subnet_ids          = [azurerm_subnet.cicd_runner.id]
  os_type             = "Linux"

  container {
    name   = "runner"
    image  = "ghcr.io/actions/actions-runner:latest"
    cpu    = "2.0"
    memory = "4.0"

    environment_variables = {
      GITHUB_URL   = "https://github.com/${var.github_org}/${var.github_repo}"
      RUNNER_TOKEN = var.github_runner_token
    }
  }
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# Get credentials (must run from within the VNet or connected network)
az aks get-credentials \
  --resource-group <rg> \
  --name <cluster-name>

# Use Bastion to connect to jumpbox, then run:
kubectl get nodes

# Run commands directly from Azure CLI (uses private endpoint)
az aks command invoke \
  --resource-group <rg> \
  --name <cluster-name> \
  --command "kubectl get nodes"
```

## Conclusion

The `az aks command invoke` command allows running kubectl commands against private clusters from anywhere with Azure CLI access—it routes through the Azure control plane without requiring network connectivity to the private endpoint. This is useful for emergency access and CI/CD pipelines that can't reach the private VNet. For ongoing CI/CD, deploy self-hosted GitHub Actions runners or Azure DevOps agents within the VNet. Using `private_dns_zone_id = "System"` is the simplest configuration—Azure manages the private DNS zone automatically; use a custom zone only when you need to share it with multiple peered VNets.
