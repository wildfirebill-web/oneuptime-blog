# How to Deploy Portainer on Azure Virtual Machines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Azure, Virtual Machine, Docker, Cloud, Self-Hosted, DevOps

Description: Deploy Portainer on an Azure Virtual Machine with Network Security Group rules, managed disks, and optional Azure Container Registry integration.

## Introduction

Azure Virtual Machines provide a reliable platform for running Portainer in the cloud. With Azure's Network Security Groups, Managed Disks, and integration with Azure Container Registry (ACR), you can build a production-ready container management environment. This guide covers VM creation, security configuration, and Azure-specific integrations.

## Prerequisites

- Azure subscription with VM creation permissions
- Azure CLI installed (optional)
- SSH key pair

## Step 1: Create the Virtual Machine

### Via Azure Portal

1. Navigate to **Virtual Machines > Create**
2. Configure:
   - **Subscription**: Your subscription
   - **Resource Group**: Create new: `portainer-rg`
   - **Name**: `portainer-vm`
   - **Region**: Choose closest region
   - **Image**: Ubuntu Server 24.04 LTS
   - **Size**: `Standard_B2s` (2 vCPU, 4GB RAM) minimum
3. Under **Administrator account**, select **SSH public key**
4. Upload or paste your public key
5. Under **Inbound port rules**, select **SSH (22)**
6. Click **Review + create**

### Via Azure CLI

```bash
# Login to Azure

az login

# Create resource group
az group create \
  --name portainer-rg \
  --location eastus

# Create VM
az vm create \
  --resource-group portainer-rg \
  --name portainer-vm \
  --image Ubuntu2404 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard
```

## Step 2: Configure Network Security Group

### Via Portal

1. Go to your VM's **Networking** settings
2. Click **Add inbound port rule**
3. Add rules:
   - Priority 310: TCP 9000 from your IP
   - Priority 320: TCP 9443 from your IP
   - Priority 330: TCP 80 from Internet (if needed)
   - Priority 340: TCP 443 from Internet (if needed)

### Via Azure CLI

```bash
# Get your public IP
MY_IP=$(curl -s https://checkip.amazonaws.com)

# Add Portainer HTTP rule
az network nsg rule create \
  --resource-group portainer-rg \
  --nsg-name portainer-vmNSG \
  --name AllowPortainer9000 \
  --protocol tcp \
  --priority 310 \
  --destination-port-range 9000 \
  --source-address-prefixes ${MY_IP}/32 \
  --access allow

# Add Portainer HTTPS rule
az network nsg rule create \
  --resource-group portainer-rg \
  --nsg-name portainer-vmNSG \
  --name AllowPortainer9443 \
  --protocol tcp \
  --priority 320 \
  --destination-port-range 9443 \
  --source-address-prefixes ${MY_IP}/32 \
  --access allow
```

## Step 3: Install Docker on the VM

```bash
# SSH to the VM
ssh azureuser@<vm-public-ip>

# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker azureuser
newgrp docker

sudo systemctl enable docker
```

## Step 4: Attach a Managed Data Disk

```bash
# Via Azure CLI, attach a 64GB Premium SSD
az vm disk attach \
  --resource-group portainer-rg \
  --vm-name portainer-vm \
  --name portainer-data-disk \
  --size-gb 64 \
  --sku Premium_LRS \
  --new

# On the VM, format and mount
sudo fdisk -l  # Find the new disk (usually /dev/sdc)
sudo mkfs.ext4 /dev/sdc
sudo mkdir -p /data
echo '/dev/sdc /data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
sudo mount -a

# Configure Docker to use data disk
sudo mkdir -p /data/docker
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "data-root": "/data/docker"
}
EOF
sudo systemctl restart docker
```

## Step 5: Deploy Portainer

```bash
docker volume create portainer_data

docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 6: Integrate with Azure Container Registry

```bash
# Login to ACR
az acr login --name yourregistryname

# Get credentials for Portainer
az acr credential show --name yourregistryname
```

In Portainer, add ACR as a registry:
1. Navigate to **Registries > Add registry**
2. Select **Azure**
3. Enter your ACR URL: `yourregistryname.azurecr.io`
4. Enter username and password from the credentials output

## Step 7: Configure Auto-Shutdown (Cost Savings)

For development VMs, configure auto-shutdown:

```bash
az vm auto-shutdown \
  --resource-group portainer-rg \
  --name portainer-vm \
  --time 2300 \
  --email yourname@example.com
```

## Conclusion

Portainer on Azure VM provides a straightforward cloud container management solution with familiar Azure security controls. Managed Disks ensure data persistence and support snapshots, while Azure Container Registry integration enables private image management. Network Security Groups keep Portainer access restricted to authorized sources.
