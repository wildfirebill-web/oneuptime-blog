# How to Deploy Containers to Azure ACI via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Azure, ACI, Container Instances, Cloud

Description: Learn how to deploy containers to Azure Container Instances (ACI) using Portainer as a management interface.

## What Is Azure Container Instances?

Azure Container Instances (ACI) is a serverless container service that runs containers without managing virtual machines. You pay per second of execution. ACI is ideal for:

- Short-lived batch jobs
- Event-driven workloads
- Burst capacity for Kubernetes clusters
- Quick demos and testing

## Connecting Portainer to Azure ACI

Portainer can manage ACI deployments using Docker's Azure context integration:

### Step 1: Install Docker ACI Context on Portainer Server

```bash
# Install the Docker ACI integration (on the Portainer host)
docker context create aci my-aci-context \
  --subscription-id <your-subscription-id> \
  --resource-group <your-resource-group> \
  --location eastus

# Verify the context was created
docker context ls
```

### Step 2: Add the ACI Environment in Portainer

1. In Portainer, go to **Environments > Add environment**.
2. Select **Azure ACI** as the environment type.
3. Enter your Azure credentials:
   - **Subscription ID**
   - **Resource group**
   - **Region**
   - **Client ID and Secret** (from an Azure Service Principal)
4. Click **Connect**.

## Deploying a Container to ACI via Portainer

Once the ACI environment is connected:

1. Navigate to the ACI environment.
2. Go to **Containers > Add container**.
3. Configure:
   - **Image**: Container image (e.g., `nginx:alpine`).
   - **CPU and memory**: Resource allocation.
   - **Ports**: Port mappings.
   - **Environment variables**: Configuration.
4. Click **Deploy the container**.

## Equivalent Azure CLI Deployment

```bash
# Deploy a container to ACI via CLI
az container create \
  --resource-group my-resource-group \
  --name my-container \
  --image nginx:alpine \
  --cpu 1 \
  --memory 1 \
  --ports 80 \
  --ip-address public \
  --location eastus

# Check deployment status
az container show \
  --resource-group my-resource-group \
  --name my-container \
  --query "{status:instanceView.state, fqdn:ipAddress.fqdn}"

# View container logs
az container logs \
  --resource-group my-resource-group \
  --name my-container
```

## ACI with a Custom Registry

```bash
# Deploy from a private registry
az container create \
  --resource-group my-resource-group \
  --name my-app \
  --image registry.mycompany.com/my-app:latest \
  --registry-login-server registry.mycompany.com \
  --registry-username myuser \
  --registry-password mypassword \
  --cpu 1 \
  --memory 2
```

## Cost Considerations

ACI pricing is per second:
- vCPU: ~$0.0000135/second (~$1.17/day for 1 vCPU)
- Memory: ~$0.0000015/GB-second

For workloads running continuously, ACI becomes more expensive than a small VM. Use ACI for:
- Workloads running < 8 hours/day
- Bursty or event-driven processing
- Development and testing

## Conclusion

Portainer's Azure ACI integration lets you manage serverless containers alongside your Swarm and Kubernetes workloads from a single interface. ACI is best for sporadic, short-lived workloads where you want to avoid VM overhead.
