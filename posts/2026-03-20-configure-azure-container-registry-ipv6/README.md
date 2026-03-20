# How to Configure Azure Container Registry with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure Container Registry, ACR, IPv6, Azure, Docker, AKS, DevOps

Description: Configure Azure Container Registry to push and pull container images over IPv6, using Azure Private Endpoints with dual-stack support and AKS IPv6 integration.

---

Azure Container Registry (ACR) supports IPv6 through dual-stack private endpoints and Azure's IPv6-capable global infrastructure. This guide covers configuring ACR for IPv6 access from both Azure and external environments.

## Checking ACR IPv6 Availability

```bash
# Check if ACR endpoint resolves to IPv6

dig AAAA myregistry.azurecr.io +short

# Test connectivity
curl -6 https://myregistry.azurecr.io/v2/ 2>&1 | head -5

# Check Azure login endpoint
dig AAAA login.microsoftonline.com +short
```

## Installing Azure CLI and Authenticating

```bash
# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login to Azure
az login

# Or use service principal authentication
az login --service-principal \
  --username APP_ID \
  --password PASSWORD \
  --tenant TENANT_ID

# Configure Docker for ACR
az acr login --name myregistry
```

## Creating an ACR with Premium SKU (for Private Endpoints)

```bash
# Create Resource Group
az group create \
  --name myResourceGroup \
  --location eastus

# Create ACR (Premium required for private endpoints)
az acr create \
  --resource-group myResourceGroup \
  --name myregistry \
  --sku Premium \
  --location eastus

# Verify creation
az acr show --name myregistry --query loginServer
```

## Configuring ACR Private Endpoint with IPv6

```bash
# Create a VNet with IPv6 support
az network vnet create \
  --resource-group myResourceGroup \
  --name myVNet \
  --address-prefixes 10.0.0.0/16 2001:db8::/32 \
  --subnet-name mySubnet \
  --subnet-prefixes 10.0.1.0/24 2001:db8::1:0/112

# Create private endpoint for ACR
ACR_ID=$(az acr show --name myregistry --query id -o tsv)

az network private-endpoint create \
  --resource-group myResourceGroup \
  --name myACRPrivateEndpoint \
  --vnet-name myVNet \
  --subnet mySubnet \
  --private-connection-resource-id "$ACR_ID" \
  --group-id registry \
  --connection-name myACRConnection

# Check the private endpoint has IPv6 addresses
az network private-endpoint show \
  --name myACRPrivateEndpoint \
  --resource-group myResourceGroup \
  --query customDnsConfigs
```

## Pushing and Pulling Images over IPv6

```bash
# Login to ACR
az acr login --name myregistry

# Build and push image
docker build -t myapp:latest .
docker tag myapp:latest myregistry.azurecr.io/myapp:latest
docker push myregistry.azurecr.io/myapp:latest

# List repositories
az acr repository list --name myregistry

# Pull image
docker pull myregistry.azurecr.io/myapp:latest

# Test over IPv6 explicitly
curl -6 -u "myregistry:$(az acr credential show -n myregistry --query passwords[0].value -o tsv)" \
  https://myregistry.azurecr.io/v2/myapp/tags/list
```

## Configuring AKS with IPv6 to Use ACR

```bash
# Create AKS cluster with IPv6 and ACR integration
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --attach-acr myregistry \
  --ip-families IPv4 IPv6 \
  --pod-cidr 10.244.0.0/16 \
  --pod-cidr-v6 fd12:3456:789a::/48 \
  --network-plugin azure \
  --enable-managed-identity

# Get credentials
az aks get-credentials --resource-group myResourceGroup \
  --name myAKSCluster

# Deploy using ACR image
kubectl create deployment myapp \
  --image=myregistry.azurecr.io/myapp:latest
```

## Geo-Replication for IPv6 Availability

```bash
# Add geo-replication to another Azure region
az acr replication create \
  --registry myregistry \
  --location westeurope

# List replications
az acr replication list --registry myregistry --output table
```

## ACR Webhook for IPv6 Events

```bash
# Create a webhook for push events (sent over HTTPS, IPv6 compatible)
az acr webhook create \
  --registry myregistry \
  --name mywebhook \
  --uri https://[2001:db8::webhook]:8080/webhook \
  --actions push delete

# Test the webhook
az acr webhook ping --registry myregistry --name mywebhook
```

Azure Container Registry integrates well with IPv6 infrastructure, particularly through private endpoints with dual-stack support, enabling secure image management in Azure's IPv6-capable network fabric.
