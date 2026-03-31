# How to Configure Drone CI with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Drone CI, CI/CD, Docker, DevOps, Pipeline

Description: Configure Drone CI server and runners to support IPv6 networking, enable IPv6 in pipeline containers, and test IPv6 connectivity in Drone pipeline steps.

## Introduction

Drone CI is a container-native CI/CD platform where every pipeline step runs in a Docker container. Enabling IPv6 requires configuring Docker for IPv6 on the Drone runner hosts and optionally the Drone server itself.

## Step 1: Configure Drone Server with IPv6

```yaml
# docker-compose.yml - Drone server with IPv6 binding

version: "3.8"

services:
  drone:
    image: drone/drone:2
    container_name: drone
    environment:
      DRONE_GITHUB_CLIENT_ID: <github-client-id>
      DRONE_GITHUB_CLIENT_SECRET: <github-client-secret>
      DRONE_RPC_SECRET: <shared-secret>
      DRONE_SERVER_HOST: drone.example.com
      DRONE_SERVER_PROTO: https
    ports:
      # Bind to both IPv4 and IPv6
      - "[::]:80:80"
      - "[::]:443:443"
    volumes:
      - drone-data:/data
    networks:
      - drone-net

networks:
  drone-net:
    enable_ipv6: true
    ipam:
      config:
        - subnet: "2001:db8:drone::/64"

volumes:
  drone-data:
```

## Step 2: Configure Drone Runner with Docker IPv6

```yaml
# docker-compose.runner.yml - Drone Docker runner with IPv6

version: "3.8"

services:
  drone-runner:
    image: drone/drone-runner-docker:1
    container_name: drone-runner
    environment:
      DRONE_RPC_PROTO: https
      DRONE_RPC_HOST: drone.example.com
      DRONE_RPC_SECRET: <shared-secret>
      DRONE_RUNNER_CAPACITY: 4
      DRONE_RUNNER_NAME: ipv6-runner
      # Pass Docker IPv6 config to pipeline containers
      DRONE_RUNNER_VOLUMES: /etc/docker/daemon.json:/etc/docker/daemon.json
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
```

## Step 3: Docker IPv6 Configuration on Runner Host

```bash
# Configure Docker on the Drone runner host for IPv6

sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "ipv6": true,
  "fixed-cidr-v6": "fd12:3456:789a::/64",
  "ip6tables": true,
  "experimental": true
}
EOF

sudo systemctl restart docker

# Verify Docker IPv6

docker run --rm busybox ip -6 addr
```

## Step 4: Write a Drone Pipeline with IPv6 Steps

```yaml
# .drone.yml

kind: pipeline
type: docker
name: ipv6-pipeline

steps:
  - name: verify-ipv6
    image: ubuntu:22.04
    commands:
      # Install networking tools
      - apt-get update -q && apt-get install -y iputils-ping curl iproute2 -q
      # Check IPv6 address in container
      - ip -6 addr show
      # Test IPv6 connectivity (internal network)
      - ping6 -c 2 fd12:3456:789a::1 || echo "Link-local IPv6 only"

  - name: run-tests
    image: python:3.12
    environment:
      APP_IPV6_ENABLED: "true"
    commands:
      - pip install -q pytest requests
      - python -m pytest tests/ -v
    depends_on:
      - verify-ipv6

  - name: build-docker-image
    image: plugins/docker
    settings:
      repo: myregistry.example.com/myapp
      tags:
        - ${DRONE_COMMIT_SHA:0:8}
        - latest
      # Use IPv6 for docker push if registry supports it
      registry: myregistry.example.com
    depends_on:
      - run-tests
    when:
      branch:
        - main

---
# Pipeline with service containers (e.g., IPv6 test server)
kind: pipeline
type: docker
name: ipv6-integration

services:
  - name: test-nginx
    image: nginx:latest
    ports:
      - 80

steps:
  - name: wait-for-services
    image: busybox
    commands:
      - sleep 5

  - name: test-service-ipv6
    image: curlimages/curl:latest
    commands:
      # Drone services are reachable by name on the pipeline network
      # Check if the service network has IPv6
      - ip -6 addr show
      # Test HTTP connectivity to service
      - curl -v http://test-nginx/
```

## Step 5: IPv6 Network Configuration for Drone Pipelines

Drone uses Docker networks to connect pipeline steps. To configure IPv6 for the pipeline network:

```yaml
# .drone.yml with custom network configuration

kind: pipeline
type: docker
name: ipv6-network-test

platform:
  os: linux
  arch: amd64

# Note: Drone doesn't support custom network config natively
# Use a setup step to configure the environment
steps:
  - name: setup-ipv6
    image: ubuntu:22.04
    privileged: true  # Required for network configuration
    commands:
      # Enable IPv6 sysctls in the container
      - sysctl -w net.ipv6.conf.all.disable_ipv6=0 || true
      - ip -6 addr show

  - name: ipv6-test
    image: python:3.12
    commands:
      - python3 -c "import socket; print(socket.has_ipv6)"
      - python3 -c "
import socket
s = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
s.bind(('::', 0))
print('IPv6 socket binding OK:', s.getsockname())
s.close()
"
```

## Verifying Drone Pipeline IPv6

```bash
# Check Drone runner logs for IPv6-related issues
docker logs drone-runner 2>&1 | grep -i ipv6

# Inspect a running pipeline container
docker ps | grep drone
CONTAINER_ID=$(docker ps | grep drone-pipeline | awk '{print $1}' | head -1)
docker exec $CONTAINER_ID ip -6 addr show

# Check the network used by pipeline containers
docker network ls | grep drone
docker network inspect <drone-network-id> | python3 -m json.tool | grep -A10 IPAM
```

## Conclusion

Drone CI's container-native architecture means IPv6 support is primarily a Docker configuration concern. Enabling IPv6 in Docker daemon on the runner host propagates IPv6 addressing to pipeline containers. The `.drone.yml` pipeline can then test IPv6 connectivity in steps, build IPv6-capable Docker images, and deploy to IPv6 infrastructure. For service containers, the pipeline network automatically provides IPv6 addresses when the Docker daemon is configured for IPv6.
