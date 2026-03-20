# How to Hide Specific Containers from Portainer Using Labels

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Labels, Configuration, Access Control

Description: Learn how to use Docker labels to hide specific containers from appearing in the Portainer UI for cleaner views and access management.

---

Portainer supports a label-based mechanism to exclude specific containers from appearing in its UI. This is useful for hiding infrastructure containers (like Portainer itself, monitoring agents, or log shippers) from regular users.

## The Hide Label

Add the following label to any container you want hidden from Portainer:

```
io.portainer.agent.hide=true
```

Alternatively, for the blacklisted label setting:
```
com.docker.compose.oneoff=true
```

## Method 1: Docker Run with Labels

```bash
# Start a container with the hide label
# This container will not appear in Portainer's container list
docker run -d \
  --name my-agent \
  --label "io.portainer.agent.hide=true" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  my-monitoring-agent:latest
```

## Method 2: Docker Compose Labels

```yaml
# docker-compose.yml
version: "3.8"

services:
  app:
    image: myapp:latest
    # This container WILL show in Portainer

  log-shipper:
    image: fluent/fluent-bit:latest
    # Hide this infrastructure container from Portainer UI
    labels:
      - "io.portainer.agent.hide=true"
    volumes:
      - /var/log:/var/log:ro

  metrics-agent:
    image: prom/node-exporter:latest
    # Also hidden from Portainer
    labels:
      - "io.portainer.agent.hide=true"
    network_mode: host
```

## Method 3: Configure Blacklisted Labels in Portainer Settings

In Portainer, you can define labels whose presence causes containers to be hidden globally. This is configured server-side rather than requiring labels on every container:

```bash
# Authenticate
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Configure blacklisted labels in Portainer settings
# Any container with these labels will be hidden
curl -X PUT \
  https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "BlackListedLabels": [
      {"name": "hide-from-portainer", "value": "true"},
      {"name": "infrastructure-only", "value": "true"}
    ]
  }' \
  --insecure
```

Then add the matching labels to containers you want hidden:

```bash
# Start Portainer Agent hidden from Portainer UI
docker run -d \
  --name portainer_agent \
  --label "hide-from-portainer=true" \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  portainer/agent:latest
```

## Hiding Portainer Itself

A common use case is hiding the Portainer container from its own interface:

```bash
# Recreate Portainer with a hide label
docker stop portainer && docker container rm portainer

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  --label "hide-from-portainer=true" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

---

*Build clean, user-friendly container management and monitor your infrastructure with [OneUptime](https://oneuptime.com).*
