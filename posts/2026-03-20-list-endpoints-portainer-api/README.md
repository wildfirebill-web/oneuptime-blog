# How to List All Endpoints via the Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Endpoints, Automation, REST API

Description: Learn how to list and filter all Portainer environments (endpoints) using the REST API for automation and scripting.

## Overview

In Portainer's API, "endpoints" refer to what the UI calls "environments" — your connected Docker, Kubernetes, and Swarm targets. The `/api/endpoints` endpoint lets you query them programmatically.

## Basic Listing

```bash
# List all endpoints
curl -s "https://portainer.mycompany.com/api/endpoints" \
  -H "Authorization: Bearer ${API_TOKEN}" | jq '.'
```

## Response Structure

```json
[
  {
    "Id": 1,
    "Name": "local-docker",
    "Type": 1,
    "URL": "unix:///var/run/docker.sock",
    "GroupId": 1,
    "Status": 1,
    "Snapshots": [...],
    "Tags": []
  },
  {
    "Id": 2,
    "Name": "production-k8s",
    "Type": 7,
    "URL": "https://k8s-api.mycompany.com:6443",
    "GroupId": 2,
    "Status": 1
  }
]
```

## Endpoint Type Values

| Type | Description |
|------|-------------|
| 1 | Docker standalone (local socket) |
| 2 | Docker standalone (agent) |
| 3 | Azure ACI |
| 4 | Docker Swarm via agent |
| 5 | Kubernetes via kubeconfig |
| 6 | Docker Edge agent |
| 7 | Kubernetes via agent |

## Filtering Endpoints

```bash
# Get only Kubernetes environments
curl -s "https://portainer.mycompany.com/api/endpoints" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | select(.Type == 7 or .Type == 5)]'

# Get endpoints by name
curl -s "https://portainer.mycompany.com/api/endpoints" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | select(.Name | contains("production"))]'

# Get only online endpoints (Status == 1)
curl -s "https://portainer.mycompany.com/api/endpoints" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | select(.Status == 1)] | {count: length, names: [.[].Name]}'
```

## Getting a Specific Endpoint by ID

```bash
# Get details of endpoint with ID 2
curl -s "https://portainer.mycompany.com/api/endpoints/2" \
  -H "Authorization: Bearer ${API_TOKEN}" | jq '.'
```

## Using Endpoint IDs in Other API Calls

Most Portainer API calls require an endpoint ID. Extract it for use in subsequent calls:

```bash
#!/bin/bash
# Get the ID of an endpoint by name

ENDPOINT_NAME="production-k8s"

ENDPOINT_ID=$(curl -s "https://portainer.mycompany.com/api/endpoints" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq --arg name "$ENDPOINT_NAME" '.[] | select(.Name == $name) | .Id')

echo "Endpoint ID for ${ENDPOINT_NAME}: ${ENDPOINT_ID}"

# Use in a subsequent API call (e.g., list stacks in that endpoint)
curl -s "https://portainer.mycompany.com/api/stacks?filters=%7B%22EndpointID%22:${ENDPOINT_ID}%7D" \
  -H "Authorization: Bearer ${API_TOKEN}" | jq '[.[] | .Name]'
```

## Pagination

For large Portainer installations:

```bash
# Use limit and start parameters for pagination
curl -s "https://portainer.mycompany.com/api/endpoints?start=1&limit=10" \
  -H "Authorization: Bearer ${API_TOKEN}" | jq '.'
```

## Conclusion

The `/api/endpoints` endpoint is the starting point for most Portainer API automation. Get the environment ID first, then use it in all subsequent calls for stack deployments, container management, and more.
