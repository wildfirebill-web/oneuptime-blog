# How to Select Azure Regions for ACI Deployments in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Azure, ACI, Regions, Cloud Infrastructure

Description: Learn how to choose the appropriate Azure region for Azure Container Instances deployments in Portainer for performance and compliance.

## Why Region Selection Matters

Choosing the right Azure region for ACI deployments affects:

- **Latency**: Deploy close to your users or downstream services.
- **Compliance**: Data residency laws may require specific geographic regions.
- **Availability**: Not all Azure services and VM sizes are available in every region.
- **Cost**: Pricing varies slightly by region.

## Regions Available for ACI

```bash
# List all regions where ACI is available
az provider show -n Microsoft.ContainerInstance \
  --query "resourceTypes[?resourceType=='containerGroups'].locations" \
  -o table
```

Common ACI-supported regions:
- `eastus`, `eastus2` — US East Coast
- `westus`, `westus2`, `westus3` — US West Coast
- `northeurope`, `westeurope` — Europe
- `eastasia`, `southeastasia` — Asia Pacific
- `australiaeast` — Australia

## Selecting a Region When Adding the ACI Environment in Portainer

When connecting an ACI environment:

1. Go to **Environments > Add environment > Azure ACI**.
2. Enter your subscription and resource group details.
3. In the **Region** dropdown, select your target Azure region.
4. Click **Connect**.

The region selection determines where new container groups are deployed.

## Deploying to a Specific Region via CLI

```bash
# Specify region when creating a container
az container create \
  --resource-group my-resource-group \
  --name my-app \
  --image nginx:alpine \
  --location westeurope \   # Target region
  --cpu 1 \
  --memory 1 \
  --ports 80

# List container groups per region
az container list \
  --resource-group my-resource-group \
  --query "[?location=='westeurope'].{name:name, status:instanceView.state}"
```

## Multi-Region ACI Strategy

For high availability or geographic distribution, deploy containers in multiple regions:

```bash
# Deploy to US East
az container create \
  --resource-group rg-east \
  --name my-app-east \
  --image my-app:latest \
  --location eastus \
  --cpu 1 --memory 1

# Deploy to West Europe
az container create \
  --resource-group rg-europe \
  --name my-app-europe \
  --image my-app:latest \
  --location westeurope \
  --cpu 1 --memory 1

# Use Azure Traffic Manager or Front Door to route between regions
```

## Checking Region-Specific ACI Capabilities

Not all container configurations are available in every region:

```bash
# Check if GPU containers are available in a region
az container show -h  # Check supported GPU SKUs by region

# Check availability zone support for ACI
az provider show -n Microsoft.ContainerInstance \
  --query "resourceTypes[?resourceType=='containerGroups'].zoneMappings"
```

## Region Selection Checklist

Before selecting a region:
- [ ] Is the region within your required data residency boundary?
- [ ] Is the region close enough to your users or services for acceptable latency?
- [ ] Does the region support the container size (CPU/memory) you need?
- [ ] Does the region have a paired region for disaster recovery?

## Conclusion

Region selection for ACI in Portainer is straightforward — configure it once when adding the environment. For production workloads with compliance requirements, always verify data residency rules before deploying to a region.
