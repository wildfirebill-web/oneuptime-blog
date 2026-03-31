# How to Select Azure Regions for ACI Deployments in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Azure, ACI, Cloud, Region

Description: Learn how to choose the right Azure regions for deploying containers via Portainer's ACI integration, considering latency, data residency, and cost implications.

## Introduction

When deploying containers to Azure Container Instances through Portainer, selecting the right Azure region is a critical decision that affects latency, data residency compliance, availability, and cost. This guide covers how to select regions in the Portainer ACI interface and the considerations for making the right choice.

## Prerequisites

- Portainer BE with Azure ACI environment configured
- Understanding of your application's geographic requirements
- Azure subscription with ACI available in target regions

## Available Azure Regions for ACI

Not all Azure regions support ACI. As of 2026, commonly available ACI regions include:

```text
Americas:
  - eastus (East US - Virginia)
  - eastus2 (East US 2 - Virginia)
  - westus (West US - California)
  - westus2 (West US 2 - Washington)
  - centralus (Central US - Iowa)
  - canadacentral (Canada Central - Toronto)
  - brazilsouth (Brazil South - São Paulo)

Europe:
  - westeurope (West Europe - Netherlands)
  - northeurope (North Europe - Ireland)
  - uksouth (UK South - London)
  - francecentral (France Central - Paris)
  - germanywestcentral (Germany West Central - Frankfurt)
  - swedencentral (Sweden Central)

Asia Pacific:
  - eastasia (East Asia - Hong Kong)
  - southeastasia (Southeast Asia - Singapore)
  - australiaeast (Australia East - Sydney)
  - japaneast (Japan East - Tokyo)
  - koreacentral (Korea Central - Seoul)
  - centralindia (Central India - Pune)
```

## Step 1: Check ACI Availability in a Region

```bash
# Log into Azure CLI

az login

# Check ACI availability in all regions
az provider show \
  --namespace Microsoft.ContainerInstance \
  --query "resourceTypes[?resourceType=='containerGroups'].locations[]" \
  --output tsv | sort

# Check if a specific region supports ACI
az container list \
  --resource-group dummy-rg \
  --output table 2>/dev/null || echo "Access error (expected if RG doesn't exist)"

# List available container instance locations
az account list-locations \
  --query "[?metadata.regionCategory=='Recommended'].{Name:name, DisplayName:displayName}" \
  --output table
```

## Step 2: Select a Region When Deploying in Portainer

When creating a new container in the ACI environment:

1. In Portainer, navigate to your ACI environment.
2. Click **Add container**.
3. In the **Region** dropdown, you will see all Azure regions where ACI is available.
4. Select the region closest to your users or that meets your data residency requirements.

## Step 3: Region Selection Criteria

### Latency

Deploy close to your users or downstream services:

```bash
# Test latency to Azure regions (use Azure Speed Test or similar)
# US East (Virginia): ~10ms from US East Coast
# West Europe: ~5ms from Western Europe
# Southeast Asia: ~5ms from Singapore

# Quick latency test using curl
for REGION in eastus westeurope southeastasia; do
  echo -n "$REGION: "
  curl -s -o /dev/null -w "%{time_total}s\n" \
    "https://${REGION}.azurecontainer.io" 2>/dev/null || echo "unavailable"
done
```

### Data Residency

For compliance requirements:

```bash
# European data residency options
EU_REGIONS=("westeurope" "northeurope" "uksouth" "francecentral" "germanywestcentral")

# Check which regions support your compliance needs
# GDPR: Use EU regions (westeurope, northeurope, etc.)
# UK Data Protection: Use uksouth or ukwest
# Australian privacy law: Use australiaeast or australiasoutheast
```

### Cost Comparison

ACI pricing varies by region. Generally:

```text
Typically lowest cost: East US, West US 2
Higher cost: Australia, Brazil, Japan
```

```bash
# Query Azure pricing API for ACI costs per region
curl -s "https://prices.azure.com/api/retail/prices?api-version=2023-01-01&\$filter=serviceName eq 'Container Instances' and priceType eq 'Consumption'" | \
  jq '.Items[] | select(.armRegionName | contains("eastus")) | {region: .armRegionName, sku: .skuName, price: .retailPrice}' | head -20
```

## Step 4: Deploy to Multiple Regions for Redundancy

Use Portainer's API to deploy the same container to multiple regions:

```bash
#!/bin/bash
# deploy-multi-region.sh - Deploy container to multiple ACI regions

PORTAINER_URL="https://portainer.example.com"
ACI_ENDPOINT=5  # Your ACI endpoint ID
REGIONS=("eastus" "westeurope" "southeastasia")

TOKEN=$(curl -s -X POST "${PORTAINER_URL}/api/auth" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' | jq -r '.jwt')

for REGION in "${REGIONS[@]}"; do
  echo "Deploying to $REGION..."

  curl -s -X POST -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "${PORTAINER_URL}/api/endpoints/${ACI_ENDPOINT}/azure/aci" \
    -d "{
      \"location\": \"$REGION\",
      \"name\": \"myapp-${REGION}\",
      \"resourceGroup\": \"portainer-aci-rg\",
      \"containers\": [{
        \"name\": \"myapp\",
        \"image\": \"myapp:latest\",
        \"resources\": {\"requests\": {\"cpu\": 0.5, \"memoryInGB\": 1.0}},
        \"ports\": [{\"port\": 80, \"protocol\": \"TCP\"}]
      }],
      \"osType\": \"Linux\",
      \"ipAddress\": {
        \"type\": \"Public\",
        \"ports\": [{\"port\": 80, \"protocol\": \"TCP\"}],
        \"dnsNameLabel\": \"myapp-${REGION}\"
      }
    }"

  echo "Deployed to $REGION: myapp-${REGION}.${REGION}.azurecontainer.io"
done
```

## Step 5: Use Azure Traffic Manager for Global Load Balancing

When running in multiple regions, use Azure Traffic Manager to route users to the closest deployment:

```bash
# Create Traffic Manager profile
az network traffic-manager profile create \
  --name myapp-global \
  --resource-group portainer-aci-rg \
  --routing-method Performance \
  --unique-dns-name myapp-global

# Add endpoints for each region
for REGION in eastus westeurope southeastasia; do
  az network traffic-manager endpoint create \
    --name "endpoint-${REGION}" \
    --profile-name myapp-global \
    --resource-group portainer-aci-rg \
    --type externalEndpoints \
    --target "myapp-${REGION}.${REGION}.azurecontainer.io" \
    --endpoint-location $REGION
done
```

## Conclusion

Selecting the right Azure region for ACI deployments in Portainer requires balancing latency, data residency, availability, and cost. Use the region dropdown in the Portainer ACI deployment form for single-region deployments, or script multi-region deployments through the Portainer API. Combine multi-region ACI with Azure Traffic Manager to achieve global availability and low latency for users worldwide.
