# How to Configure Azure Cloud Provider in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Azure, Cloud Provider

Description: Configure the Azure cloud provider in Rancher-managed clusters to enable Azure Load Balancers, Azure Disks, and Azure Files integration.

## Introduction

The Azure cloud provider integration lets your Rancher-managed Kubernetes clusters provision Azure Load Balancers for Services, use Azure Managed Disks for PersistentVolumes, and mount Azure Files shares. This guide covers configuring the Azure cloud provider for RKE2 clusters deployed on Azure VMs.

## Prerequisites

- Rancher managing an RKE2 cluster on Azure VMs
- An Azure Service Principal with Contributor rights to the resource group
- All VMs in the same resource group and availability set (or VMSS)

## Step 1: Create an Azure Service Principal

```bash
# Log in to Azure CLI
az login

# Create a service principal with Contributor role on the resource group
az ad sp create-for-rbac \
  --name "rancher-cloud-provider" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<resource-group>

# Output (save these values):
# {
#   "appId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",         ← client-id
#   "displayName": "rancher-cloud-provider",
#   "password": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",   ← client-secret
#   "tenant": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"         ← tenant-id
# }
```

## Step 2: Create the cloud-config.json

```json
{
  "cloud": "AzurePublicCloud",
  "tenantId": "<tenant-id>",
  "subscriptionId": "<subscription-id>",
  "aadClientId": "<service-principal-app-id>",
  "aadClientSecret": "<service-principal-password>",
  "resourceGroup": "<resource-group-name>",
  "location": "eastus",
  "subnetName": "<subnet-name>",
  "securityGroupName": "<nsg-name>",
  "vnetName": "<vnet-name>",
  "vnetResourceGroup": "<vnet-resource-group>",
  "primaryAvailabilitySetName": "<availability-set-name>",
  "cloudProviderBackoff": true,
  "cloudProviderBackoffRetries": 6,
  "cloudProviderBackoffDuration": 5,
  "cloudProviderRatelimit": true,
  "cloudProviderRateLimitQPS": 6,
  "cloudProviderRateLimitBucket": 20,
  "useManagedIdentityExtension": false,
  "loadBalancerSku": "standard"
}
```

Save this as `/etc/rancher/rke2/azure-cloud-config.json` on each node.

## Step 3: Configure RKE2 Nodes

```yaml
# /etc/rancher/rke2/config.yaml (both server and agent nodes)
cloud-provider-name: azure
cloud-provider-config: /etc/rancher/rke2/azure-cloud-config.json
```

## Step 4: Configure via Rancher UI

1. Navigate to **Cluster Management** → select the cluster → **⋮ → Edit Config**.
2. Under **Cloud Provider**, select **Azure**.
3. Fill in the fields:
   - **Tenant ID**: Azure AD Tenant ID
   - **Subscription ID**: Azure Subscription ID
   - **Client ID**: Service Principal App ID
   - **Client Secret**: Service Principal Password
   - **Resource Group**: Azure Resource Group
   - **Location**: Azure Region (e.g., `eastus`)
4. Click **Save**.

## Step 5: Install the Azure Cloud Controller Manager

```bash
# Add the Azure CCM Helm chart repo
helm repo add azure-ccm \
  https://raw.githubusercontent.com/kubernetes-sigs/cloud-provider-azure/master/helm/repo
helm repo update

# Create the cloud-config secret
kubectl create secret generic azure-cloud-config \
  --from-file=cloud-config=/etc/rancher/rke2/azure-cloud-config.json \
  -n kube-system

# Install Azure CCM
helm install azure-cloud-controller-manager azure-ccm/cloud-controller-manager \
  --namespace kube-system \
  --set cloudConfig.secretName=azure-cloud-config
```

## Step 6: Install the Azure Disk CSI Driver

```bash
# Install Azure Disk CSI Driver
helm repo add azuredisk-csi-driver \
  https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/charts
helm repo update

helm install azuredisk-csi-driver azuredisk-csi-driver/azuredisk-csi-driver \
  --namespace kube-system \
  --set controller.cloudConfigSecretName=azure-cloud-config \
  --set node.cloudConfigSecretName=azure-cloud-config
```

## Step 7: Create Azure StorageClasses

```yaml
# azure-disk-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-managed-disk
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS      # or Standard_LRS
  kind: Managed
  cachingMode: ReadOnly
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```bash
kubectl apply -f azure-disk-storageclass.yaml
```

## Step 8: Verify the Integration

```bash
# Test LoadBalancer provisioning
kubectl run nginx --image=nginx --port=80
kubectl expose pod nginx \
  --type=LoadBalancer \
  --name=azure-lb-test

# Watch for the Azure Load Balancer IP
kubectl get service azure-lb-test -w
# EXTERNAL-IP should show an Azure public IP within 1-2 minutes

# Test PVC with Azure Managed Disk
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-disk-test
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: azure-managed-disk
  resources:
    requests:
      storage: 10Gi
EOF

kubectl get pvc azure-disk-test -w
```

## Common Issues

| Issue | Resolution |
|---|---|
| `EXTERNAL-IP` stays `<pending>` | Check Service Principal permissions and subnet configuration |
| `PVC stuck in Pending` | Verify the StorageClass name and CSI driver is running |
| `AuthorizationFailed` | Service Principal lacks Contributor role on the resource group |

## Conclusion

Configuring the Azure cloud provider in Rancher enables Kubernetes-native Azure resource management. With the Azure CCM and Disk CSI Driver installed, your clusters can dynamically provision Azure Load Balancers and Managed Disks without manual Azure portal intervention. Keep your Service Principal credentials secure by storing them in a Kubernetes Secret rather than embedding them in ConfigMaps.
