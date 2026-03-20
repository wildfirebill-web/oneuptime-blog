# How to Use Rancher with Azure Marketplace

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Azure, Marketplace

Description: Deploy and manage Rancher through Azure Marketplace on AKS, including Marketplace subscription activation, AKS deployment, and Azure billing integration.

## Introduction

Azure Marketplace offers SUSE Rancher as a managed application, enabling streamlined deployment on Azure Kubernetes Service (AKS) with integrated billing through your Azure subscription. This guide walks through subscribing, deploying, and operating Rancher on AKS via Azure Marketplace.

## Prerequisites

- An Azure subscription with Owner or Contributor access
- `az`, `kubectl`, and `helm` CLIs installed
- A domain name for Rancher hostname with DNS access

## Step 1: Subscribe via Azure Marketplace

1. Navigate to [Azure Marketplace → SUSE Rancher](https://azuremarketplace.microsoft.com/marketplace/).
2. Search for **"Rancher"** and select the SUSE Rancher offer.
3. Click **Get It Now** → **Continue**.
4. Choose a plan (Bring Your Own License or pay-as-you-go).
5. Click **Create** to proceed to deployment.

## Step 2: Create an AKS Cluster for Rancher

```bash
# Set variables
RESOURCE_GROUP="rancher-management-rg"
CLUSTER_NAME="rancher-aks"
LOCATION="eastus"

# Create a resource group
az group create \
  --name "${RESOURCE_GROUP}" \
  --location "${LOCATION}"

# Create the AKS cluster
az aks create \
  --resource-group "${RESOURCE_GROUP}" \
  --name "${CLUSTER_NAME}" \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --kubernetes-version 1.29 \
  --enable-managed-identity \
  --network-plugin azure \
  --network-policy azure \
  --generate-ssh-keys

# Get AKS credentials
az aks get-credentials \
  --resource-group "${RESOURCE_GROUP}" \
  --name "${CLUSTER_NAME}" \
  --overwrite-existing

kubectl get nodes
```

## Step 3: Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

kubectl rollout status deployment/cert-manager -n cert-manager
```

## Step 4: Deploy Rancher from Azure Marketplace

The Azure Marketplace Managed Application handles the Helm deployment. Alternatively, deploy manually:

```bash
# Add Rancher Helm repo
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

# Install Rancher
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.example.com \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=admin@example.com \
  --set replicas=3

kubectl rollout status deployment/rancher -n cattle-system --timeout=10m
```

## Step 5: Configure an Azure DNS Zone

```bash
# Create an Azure DNS zone (if not existing)
az network dns zone create \
  --resource-group "${RESOURCE_GROUP}" \
  --name example.com

# Get the Rancher ingress IP
RANCHER_IP=$(kubectl get service -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Create DNS A record
az network dns record-set a add-record \
  --resource-group "${RESOURCE_GROUP}" \
  --zone-name example.com \
  --record-set-name rancher \
  --ipv4-address "${RANCHER_IP}"
```

## Step 6: Enable Azure AD Integration

Rancher supports Azure AD (Entra ID) as an identity provider:

```bash
# Create an Azure AD App Registration for Rancher
APP_NAME="Rancher-OIDC"

APP_ID=$(az ad app create \
  --display-name "${APP_NAME}" \
  --sign-in-audience AzureADMyOrg \
  --query appId -o tsv)

# Create a client secret
APP_SECRET=$(az ad app credential reset \
  --id "${APP_ID}" \
  --query password -o tsv)

echo "App ID: ${APP_ID}"
echo "App Secret: ${APP_SECRET}"
echo "Tenant ID: $(az account show --query tenantId -o tsv)"
```

In Rancher UI:
1. Navigate to **☰ → Users & Authentication → Auth Provider → Azure AD**.
2. Enter the App ID, Secret, and Tenant ID.
3. Click **Enable**.

## Step 7: Configure AKS Auto-Scaling

```bash
# Enable cluster autoscaler
az aks update \
  --resource-group "${RESOURCE_GROUP}" \
  --name "${CLUSTER_NAME}" \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10

# Update the node pool
az aks nodepool update \
  --resource-group "${RESOURCE_GROUP}" \
  --cluster-name "${CLUSTER_NAME}" \
  --name nodepool1 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10
```

## Step 8: Monitor Azure Marketplace Billing

```bash
# Check your subscription usage
az consumption usage list \
  --billing-period-name "202603" \
  --query "[?contains(productName, 'Rancher')]" \
  --output table

# Monitor AKS costs with Azure Cost Management
az costmanagement query \
  --timeframe MonthToDate \
  --type Usage \
  --scope /subscriptions/<subscription-id>
```

## Conclusion

Azure Marketplace deployment of Rancher simplifies procurement and consolidates billing within your Azure subscription. Running Rancher on AKS ensures a Microsoft-managed Kubernetes control plane, while Rancher provides multi-cluster management across all your Azure and non-Azure clusters. Azure AD integration provides seamless SSO for your team, and Azure Cost Management gives visibility into platform costs.
