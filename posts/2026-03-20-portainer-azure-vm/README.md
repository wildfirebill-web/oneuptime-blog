# How to Deploy Portainer on Azure Virtual Machines - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Azure, Virtual Machine, Docker, Cloud

Description: Learn how to deploy Portainer on an Azure Virtual Machine, configure network security groups, and access Portainer securely over HTTPS.

## Prerequisites

- Azure subscription
- Azure CLI installed, or use Azure Portal
- Basic knowledge of Azure networking

## Step 1: Create the Virtual Machine

```bash
# Login to Azure CLI

az login

# Create resource group
az group create \
  --name portainer-rg \
  --location eastus

# Create VM (Ubuntu 22.04 LTS)
az vm create \
  --resource-group portainer-rg \
  --name portainer-vm \
  --image Ubuntu2204 \
  --size Standard_B2s \    # 2 vCPU, 4GB RAM
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard
```

## Step 2: Configure Network Security Group

```bash
# Allow SSH (for setup)
az network nsg rule create \
  --resource-group portainer-rg \
  --nsg-name portainer-vmNSG \
  --name AllowSSH \
  --protocol tcp \
  --priority 1000 \
  --destination-port-range 22 \
  --access Allow

# Allow Portainer HTTPS
az network nsg rule create \
  --resource-group portainer-rg \
  --nsg-name portainer-vmNSG \
  --name AllowPortainerHTTPS \
  --protocol tcp \
  --priority 1010 \
  --destination-port-range 9443 \
  --access Allow

# Allow Portainer HTTP (for initial setup or redirect)
az network nsg rule create \
  --resource-group portainer-rg \
  --nsg-name portainer-vmNSG \
  --name AllowPortainerHTTP \
  --protocol tcp \
  --priority 1020 \
  --destination-port-range 9000 \
  --access Allow
```

## Step 3: Install Docker and Portainer

```bash
# SSH into the VM
az vm show -d -g portainer-rg -n portainer-vm --query publicIps -o tsv
ssh azureuser@<PUBLIC_IP>

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker azureuser
newgrp docker

# Install Portainer
docker volume create portainer_data

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 4: Configure a DNS Name

```bash
# Get the VM's public IP resource name
az network public-ip list \
  --resource-group portainer-rg \
  --query "[].{name:name, ip:ipAddress}" \
  --output table

# Set a DNS label
az network public-ip update \
  --resource-group portainer-rg \
  --name portainer-vmPublicIP \
  --dns-name portainer-mycompany
# Portainer accessible at: portainer-mycompany.eastus.cloudapp.azure.com
```

## Step 5: Secure with Azure Bastion (Optional)

For environments where you don't want to expose SSH publicly:

```bash
# Create Bastion for secure SSH without public IP exposure
az network bastion create \
  --name portainer-bastion \
  --resource-group portainer-rg \
  --vnet-name portainer-vmVNET \
  --public-ip-address portainer-bastion-ip
```

## Step 6: Set Up Auto-Shutdown for Cost Control

```bash
az vm auto-shutdown \
  --resource-group portainer-rg \
  --name portainer-vm \
  --time 2300 \        # 11 PM UTC
  --email your@email.com
```

## Accessing Portainer

```text
URL: https://<PUBLIC_IP>:9443
Or:  https://portainer-mycompany.eastus.cloudapp.azure.com:9443
```

Complete the initial setup to create your admin account.

## Cost Optimization

| VM Size | vCPU | RAM | Monthly Cost (approx.) |
|---------|------|-----|----------------------|
| Standard_B1s | 1 | 1GB | ~$8 |
| Standard_B2s | 2 | 4GB | ~$30 |
| Standard_B2ms | 2 | 8GB | ~$60 |

Use B-series burstable VMs for dev/test Portainer instances.

## Conclusion

Deploying Portainer on Azure VM takes about 10 minutes from Azure CLI to a running Portainer dashboard. The Azure NSG provides the firewall, and optionally Azure Bastion can eliminate the need for a public SSH port. Once set up, Portainer manages all subsequent container deployments on the VM.
