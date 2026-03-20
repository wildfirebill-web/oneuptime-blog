# How to List and Manage Environments via the Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Environments, Management, Automation

Description: Learn how to list, create, update, and delete Portainer environments (endpoints) via the REST API for automated infrastructure management.

## Introduction

Portainer environments represent managed infrastructure — Docker hosts, Kubernetes clusters, ACI instances, and Edge agents. The Portainer API lets you automate full environment lifecycle management: adding new environments when new infrastructure is provisioned, updating settings, and decommissioning environments when infrastructure is retired.

## Prerequisites

- Portainer CE or BE with admin access
- Valid admin JWT token or API access token
- `curl` and `jq` installed

## Step 1: List All Environments

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"

# List all environments
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints" | jq .

# Get a compact summary
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints" | \
  jq '[.[] | {id: .Id, name: .Name, type: .Type, status: .Status, url: .URL}]'

# Count total environments
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints" | jq 'length'
```

## Step 2: Get a Specific Environment

```bash
ENDPOINT_ID=1

# Get environment details
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}" | jq .

# Get just the essential fields
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}" | jq '{
    id: .Id,
    name: .Name,
    type: .Type,
    url: .URL,
    status: .Status,
    groupId: .GroupId,
    tags: .TagIds
  }'
```

## Step 3: Add a Docker Standalone Environment

```bash
# Add a Docker environment with TLS
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints" \
  -d '{
    "Name": "production-docker",
    "EndpointCreationType": 1,
    "URL": "tcp://192.168.1.100:2376",
    "TLS": true,
    "TLSSkipVerify": false,
    "TLSCACert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
    "TLSCert": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
    "TLSKey": "-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----",
    "GroupID": 1,
    "TagIds": [1, 2]
  }' | jq .

# Add a Docker environment over Unix socket (local)
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: multipart/form-data" \
  "${PORTAINER_URL}/api/endpoints" \
  -F "Name=local-docker" \
  -F "EndpointCreationType=1" \
  -F "URL=unix:///var/run/docker.sock" | jq .
```

## Step 4: Add a Kubernetes Environment

```bash
# Add Kubernetes environment via kubeconfig file
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -F "Name=production-k8s" \
  -F "EndpointCreationType=5" \
  -F "KubernetesDeploymentMode=1" \
  -F "TLS=false" \
  -F "kubeconfig=@/path/to/kubeconfig.yaml" \
  "${PORTAINER_URL}/api/endpoints" | jq .
```

## Step 5: Update an Environment

```bash
ENDPOINT_ID=1

# Update environment name
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}" \
  -d '{
    "Name": "production-docker-v2",
    "PublicURL": "https://docker.example.com"
  }' | jq .

# Update environment tags
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}" \
  -d '{"TagIds": [1, 3, 5]}' | jq .

# Move environment to a different group
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}" \
  -d '{"GroupID": 2}' | jq .
```

## Step 6: Manage Environment Groups

```bash
# List all environment groups
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoint_groups" | jq .

# Create a new environment group
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoint_groups" \
  -d '{
    "Name": "Cloud Environments",
    "Description": "AWS, Azure, and GCP environments",
    "AssociatedEndpoints": [1, 2, 3]
  }' | jq .

# Update a group
GROUP_ID=2
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoint_groups/${GROUP_ID}" \
  -d '{
    "Name": "Cloud Environments - Production",
    "Description": "Production cloud infrastructure"
  }' | jq .
```

## Step 7: Delete an Environment

```bash
ENDPOINT_ID=5

# Delete an environment (does NOT affect running containers/services)
curl -s -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}"

echo "Environment $ENDPOINT_ID removed from Portainer."
```

## Step 8: Sync Environment Status

```bash
#!/bin/bash
# check-environment-health.sh

PORTAINER_URL="https://portainer.example.com"
TOKEN="your-token"

ENVIRONMENTS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints")

echo "=== Environment Health Report ==="
echo ""

echo "$ENVIRONMENTS" | jq -c '.[]' | while read -r ENV; do
  ID=$(echo $ENV | jq -r '.Id')
  NAME=$(echo $ENV | jq -r '.Name')
  STATUS=$(echo $ENV | jq -r '.Status')

  if [ "$STATUS" -eq 1 ]; then
    echo "UP    [$ID] $NAME"
  else
    echo "DOWN  [$ID] $NAME  *** ALERT ***"
  fi
done
```

## Conclusion

Managing Portainer environments via the API enables full automation of infrastructure registration and decommissioning. As you provision new Docker hosts or Kubernetes clusters, add them to Portainer automatically using CI/CD pipelines or Terraform. Use environment groups and tags to organize your infrastructure, and monitor environment health through regular API status checks integrated with your alerting systems.
