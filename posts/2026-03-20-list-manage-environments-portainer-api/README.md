# How to List and Manage Environments via the Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Environments, Automation, Infrastructure

Description: Learn how to create, update, and manage Portainer environments (endpoints) using the REST API for infrastructure automation.

## Environment Management Endpoints

| Method | Endpoint | Action |
|--------|----------|--------|
| GET | `/api/endpoints` | List all environments |
| GET | `/api/endpoints/{id}` | Get specific environment |
| POST | `/api/endpoints` | Create an environment |
| PUT | `/api/endpoints/{id}` | Update an environment |
| DELETE | `/api/endpoints/{id}` | Delete an environment |

## Creating a New Docker Environment

```bash
# Add a new Docker environment via Unix socket
curl -X POST "${PORTAINER_URL}/api/endpoints" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: multipart/form-data" \
  -F "Name=my-docker-host" \
  -F "EndpointCreationType=1" \
  -F "URL=unix:///var/run/docker.sock"

# Add a remote Docker host via TCP
curl -X POST "${PORTAINER_URL}/api/endpoints" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: multipart/form-data" \
  -F "Name=remote-docker" \
  -F "EndpointCreationType=1" \
  -F "URL=tcp://192.168.1.100:2376" \
  -F "TLS=true" \
  -F "TLSCACert=@ca.pem" \
  -F "TLSCert=@cert.pem" \
  -F "TLSKey=@key.pem"
```

## Connecting via Portainer Agent

```bash
# Add an environment managed by the Portainer Agent
curl -X POST "${PORTAINER_URL}/api/endpoints" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: multipart/form-data" \
  -F "Name=prod-swarm" \
  -F "EndpointCreationType=2" \
  -F "URL=https://swarm-node:9001" \
  -F "GroupID=2" \
  -F "TagIds=[1,3]"
```

## Updating an Environment

```bash
# Update environment name and group
ENDPOINT_ID=3

curl -X PUT "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "production-swarm-v2",
    "GroupID": 2,
    "TagIds": [1, 2, 5]
  }'
```

## Updating Security Settings

```bash
# Enable/disable security features on a Kubernetes environment
curl -X PUT "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Kubernetes": {
      "Configuration": {
        "UseLoadBalancer": false,
        "UseServerMetrics": true,
        "AllowNoneIngressClass": false,
        "RestrictDefaultNamespace": true
      }
    }
  }'
```

## Listing Environment Groups

```bash
# List all environment groups (for organization)
curl -s "${PORTAINER_URL}/api/endpoint_groups" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | {id: .Id, name: .Name, endpoints: (.Endpoints | length)}]'
```

## Checking Environment Health

```bash
# Get all environments and their status
curl -s "${PORTAINER_URL}/api/endpoints" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | {name: .Name, status: (if .Status == 1 then "up" else "down" end), type: .Type}]'
```

## Deleting an Environment

```bash
# Delete an environment (does not affect the underlying infrastructure)
curl -X DELETE "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}" \
  -H "Authorization: Bearer ${API_TOKEN}"
```

## Automating Environment Discovery

```bash
#!/bin/bash
# Auto-register new Swarm workers as Portainer environments

NEW_NODE_IP="$1"
NODE_NAME="worker-$(date +%s)"

# Install Portainer agent on the new node (via SSH)
ssh "${NEW_NODE_IP}" "docker run -d -p 9001:9001 \
  --name portainer-agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest"

# Register the agent in Portainer
curl -X POST "${PORTAINER_URL}/api/endpoints" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -F "Name=${NODE_NAME}" \
  -F "EndpointCreationType=2" \
  -F "URL=tcp://${NEW_NODE_IP}:9001"

echo "Registered ${NODE_NAME} (${NEW_NODE_IP}) in Portainer"
```

## Conclusion

The Portainer environments API enables fully automated infrastructure registration. Use it to programmatically onboard new Docker hosts, configure security settings, and maintain environment metadata as part of your infrastructure-as-code workflow.
