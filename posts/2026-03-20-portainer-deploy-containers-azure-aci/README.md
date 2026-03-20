# How to Deploy Containers to Azure ACI via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Azure, ACI, Docker, Cloud

Description: Learn how to deploy containers to Azure Container Instances using Portainer Business Edition's ACI environment integration, including configuration and monitoring.

## Introduction

With Azure ACI configured as an environment in Portainer, you can deploy containerized applications to Azure's serverless container platform directly from the Portainer UI. ACI is ideal for stateless workloads, batch jobs, and microservices that don't require persistent infrastructure management.

## Prerequisites

- Portainer BE with Azure ACI environment configured
- Azure ACI environment connected and showing green in Portainer
- Container images accessible from Azure (Docker Hub, ACR, or other registries)

## Step 1: Navigate to the ACI Environment

1. Log into Portainer.
2. From the **Home** dashboard, click on your **Azure ACI** environment.
3. You will see the ACI-specific interface with container groups.

## Step 2: Deploy a New Container

1. Click **Add container** in the ACI environment view.
2. Fill in the container configuration:

### Basic Configuration

- **Container name**: `my-web-app`
- **Image**: `nginx:1.25` (or your container image)
- **OS type**: Linux or Windows

Resource Configuration

```text
CPU:    0.5 vCPU  (minimum: 0.1)
Memory: 1.5 GB    (minimum: 0.1)
```

### Port Configuration

- **Port**: `80`
- **Protocol**: `TCP`

For HTTPS:
- Add port `443` with protocol `TCP`

## Step 3: Configure Environment Variables

Add environment variables your container needs:

```text
APP_ENV=production
DATABASE_URL=postgresql://user:pass@db.example.com:5432/mydb
API_KEY=your-api-key-value
LOG_LEVEL=info
```

## Step 4: Configure Networking

ACI containers can have:

- **Public IP**: Expose a public IP address directly
- **Private networking**: Deploy into an Azure Virtual Network

For a public-facing container:
1. Enable **Public IP address**
2. Set a **DNS name label**: e.g., `myapp-prod` (results in `myapp-prod.eastus.azurecontainer.io`)
3. Map the ports you exposed above

## Step 5: Deploy via the Portainer API

For automated deployments to ACI:

```bash
# Authenticate with Portainer

TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' | jq -r '.jwt')

# Get the ACI endpoint ID
ACI_ENDPOINT=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints" | \
  jq -r '.[] | select(.Type == 3) | .Id')

echo "ACI Endpoint ID: $ACI_ENDPOINT"

# Deploy a container to ACI
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/endpoints/${ACI_ENDPOINT}/azure/aci" \
  -d '{
    "location": "eastus",
    "name": "my-web-app",
    "resourceGroup": "portainer-aci-rg",
    "containers": [
      {
        "name": "nginx",
        "image": "nginx:1.25",
        "resources": {
          "requests": {
            "cpu": 0.5,
            "memoryInGB": 1.0
          }
        },
        "ports": [
          {
            "port": 80,
            "protocol": "TCP"
          }
        ],
        "environmentVariables": [
          {
            "name": "APP_ENV",
            "value": "production"
          }
        ]
      }
    ],
    "osType": "Linux",
    "ipAddress": {
      "type": "Public",
      "ports": [
        {
          "port": 80,
          "protocol": "TCP"
        }
      ],
      "dnsNameLabel": "myapp-prod"
    }
  }'
```

## Step 6: Deploy a Multi-Container Group

ACI supports running multiple containers in the same container group (similar to a Kubernetes pod):

```bash
# Multi-container ACI deployment (app + sidecar)
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/endpoints/${ACI_ENDPOINT}/azure/aci" \
  -d '{
    "location": "eastus",
    "name": "app-with-sidecar",
    "resourceGroup": "portainer-aci-rg",
    "containers": [
      {
        "name": "app",
        "image": "myapp:latest",
        "resources": {
          "requests": {"cpu": 1.0, "memoryInGB": 2.0}
        },
        "ports": [{"port": 8080, "protocol": "TCP"}]
      },
      {
        "name": "log-collector",
        "image": "fluent/fluent-bit:latest",
        "resources": {
          "requests": {"cpu": 0.1, "memoryInGB": 0.2}
        }
      }
    ],
    "osType": "Linux",
    "ipAddress": {
      "type": "Public",
      "ports": [{"port": 8080, "protocol": "TCP"}]
    }
  }'
```

## Step 7: Monitor Running Containers

In Portainer's ACI view:

1. Click on a container group name to see details.
2. View **Status**: Running, Stopped, Failed
3. Click **Logs** to see container output.
4. View resource usage (CPU, memory).

Via Azure CLI:

```bash
# Check container group status
az container show \
  --resource-group portainer-aci-rg \
  --name my-web-app \
  --query '{status: instanceView.state, ip: ipAddress.ip}' \
  -o json

# View container logs
az container logs \
  --resource-group portainer-aci-rg \
  --name my-web-app
```

## Step 8: Stop and Delete Containers

In Portainer, select the container group and click **Stop** or **Remove**.

```bash
# Stop ACI container group
az container stop --resource-group portainer-aci-rg --name my-web-app

# Delete ACI container group
az container delete --resource-group portainer-aci-rg --name my-web-app --yes
```

## Conclusion

Deploying containers to Azure ACI via Portainer combines the simplicity of Azure's serverless container platform with Portainer's familiar management interface. Use ACI for workloads that benefit from automatic scaling to zero, and leverage Portainer's API for CI/CD pipeline integration. Monitor resource usage carefully as ACI charges per second of CPU and memory consumption.
