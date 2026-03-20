# How to Set Up Portainer High Availability

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, High Availability, Swarm, Business Edition, Infrastructure

Description: Configure Portainer Business Edition for high availability using Docker Swarm with multiple replicas to ensure continuous management capability during node failures.

## Introduction

Portainer Business Edition supports high availability (HA) deployment where multiple Portainer server instances run simultaneously behind a load balancer. If one instance fails, the others continue serving requests without interruption. This guide covers HA setup on Docker Swarm and with an external load balancer.

## Prerequisites

- Portainer Business Edition (HA requires BE)
- Docker Swarm cluster with 3+ manager nodes
- Shared storage for the Portainer data volume (NFS, GlusterFS, or similar)
- A load balancer (hardware, software, or cloud LB)

## Understanding Portainer HA Architecture

```
[Load Balancer]
      |
 _____|_____
|     |     |
[Portainer] [Portainer] [Portainer]
[Instance1] [Instance2] [Instance3]
      |           |          |
      +-----+-----+-----------+
            |
     [Shared Storage]
     (portainer.db on NFS/EFS)
```

All instances share the same BoltDB database on shared storage. The load balancer distributes requests across instances.

## Step 1: Set Up Shared Storage

```bash
# Option A: NFS server (on a dedicated storage node)
# Install NFS server
sudo apt-get install -y nfs-kernel-server

# Create export directory
mkdir -p /mnt/portainer-data
chmod 777 /mnt/portainer-data

# Configure NFS exports
echo "/mnt/portainer-data *(rw,sync,no_root_squash,no_subtree_check)" >> /etc/exports
exportfs -ra

# On each Swarm node, mount the NFS share
# /etc/fstab entry:
# nfs-server-ip:/mnt/portainer-data /mnt/portainer-data nfs defaults,_netdev 0 0

# Option B: AWS EFS (for AWS environments)
aws efs create-file-system --creation-token portainer-ha
# Mount via EFS mount helper on each node
```

## Step 2: Create a Docker Swarm Cluster

```bash
# Initialize Swarm on manager1
docker swarm init --advertise-addr manager1-ip

# Get join tokens
docker swarm join-token manager
docker swarm join-token worker

# Join additional managers (minimum 3 for HA)
# On manager2 and manager3:
docker swarm join --token <manager-token> manager1-ip:2377

# Verify cluster
docker node ls
```

## Step 3: Create Portainer HA Stack

```yaml
# portainer-ha.yml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ee:latest
    command:
      - --admin-password-file=/run/secrets/portainer-admin-password
      - --license-key=YOUR-LICENSE-KEY-HERE
      # HA requires all instances to use the same shared storage
    deploy:
      mode: replicated
      replicas: 3              # Run 3 instances
      placement:
        constraints:
          - node.role == manager  # Only on manager nodes
      update_config:
        parallelism: 1         # Update one at a time
        delay: 30s
        order: start-first     # Start new before stopping old
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    ports:
      - target: 9443
        published: 9443
        mode: host             # Use host mode for better performance
    volumes:
      - portainer_data:/data   # MUST be shared storage
      - /var/run/docker.sock:/var/run/docker.sock
    secrets:
      - portainer-admin-password
    networks:
      - portainer-net

  # Portainer Agent on all nodes
  portainer-agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    deploy:
      mode: global
    networks:
      - portainer-net
    environment:
      AGENT_CLUSTER_ADDR: tasks.portainer-agent

secrets:
  portainer-admin-password:
    external: true

networks:
  portainer-net:
    driver: overlay
    attachable: true

volumes:
  portainer_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=nfs-server-ip,rw,vers=4,hard,intr
      device: ":/mnt/portainer-data"
```

## Step 4: Deploy the HA Stack

```bash
# Create the admin password secret
echo "YourSecurePassword" | docker secret create portainer-admin-password -

# Deploy the stack
docker stack deploy -c portainer-ha.yml portainer

# Verify all replicas are running
docker service ls | grep portainer
docker service ps portainer_portainer
```

## Step 5: Configure Load Balancer

### HAProxy Configuration

```
frontend portainer_frontend
    bind *:443 ssl crt /etc/ssl/portainer.pem
    mode http
    default_backend portainer_backend

    # WebSocket support
    option http-server-close
    option forwardfor

backend portainer_backend
    mode http
    balance roundrobin
    option httpchk GET /api/status

    # Health check
    http-check expect status 200

    # Portainer instances
    server portainer1 manager1-ip:9443 check ssl verify none inter 30s
    server portainer2 manager2-ip:9443 check ssl verify none inter 30s
    server portainer3 manager3-ip:9443 check ssl verify none inter 30s

    # Sticky sessions (recommended for WebSocket)
    cookie SERVERID insert indirect nocache
    server portainer1 manager1-ip:9443 check ssl verify none cookie p1
    server portainer2 manager2-ip:9443 check ssl verify none cookie p2
    server portainer3 manager3-ip:9443 check ssl verify none cookie p3

    # Timeout for WebSocket
    timeout tunnel 3600s
```

### Nginx Load Balancer

```nginx
upstream portainer {
    # Sticky sessions via IP hash
    ip_hash;
    server manager1-ip:9443;
    server manager2-ip:9443;
    server manager3-ip:9443;
}

server {
    listen 443 ssl;
    server_name portainer.yourdomain.com;

    ssl_certificate /etc/ssl/certs/portainer.crt;
    ssl_certificate_key /etc/ssl/private/portainer.key;

    location / {
        proxy_pass https://portainer;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_buffering off;
    }
}
```

## Step 6: Test Failover

```bash
# Verify HA is working
# Get the load balancer IP
LB_IP="your-lb-ip"

# Test 1: Normal operation
curl -k https://$LB_IP/api/status

# Test 2: Stop one Portainer instance
docker service scale portainer_portainer=2

# Test 3: Verify service still available
curl -k https://$LB_IP/api/status  # Should still work

# Test 4: Restore to 3 instances
docker service scale portainer_portainer=3
```

## Step 7: Monitor HA Health

```bash
# Check all Portainer replicas are healthy
docker service ps portainer_portainer --no-trunc

# Check service logs across all replicas
docker service logs portainer_portainer --tail 20

# Monitor with watch
watch -n 10 "docker service ps portainer_portainer"
```

## Conclusion

Portainer Business Edition HA requires shared storage for all instances to access the same BoltDB database, session-aware load balancing (sticky sessions recommended for WebSocket), and a minimum of 3 replicas for true high availability. The most common production setup is 3 Portainer instances on 3 Swarm manager nodes, fronted by an HAProxy or Nginx load balancer with IP hash-based sticky sessions.
