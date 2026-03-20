# How to Run Portainer on a Raspberry Pi Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Raspberry Pi, Docker Swarm, Cluster, Self-Hosted, Home Lab

Description: Set up a Raspberry Pi cluster running Docker Swarm and deploy Portainer to manage multi-node container workloads with high availability.

## Introduction

A cluster of Raspberry Pis running Docker Swarm gives you a highly available container platform at low cost. With Portainer managing the Swarm, you get a visual interface for deploying services, monitoring nodes, and managing the cluster state. This guide covers setting up a 3-node Raspberry Pi Swarm with Portainer.

## Prerequisites

- 3 or more Raspberry Pi 4 (4GB or 8GB) with Raspberry Pi OS 64-bit
- Gigabit switch and Ethernet cables
- Static IPs assigned to each Pi
- SSH access to all nodes

## Step 1: Assign Static IPs

On each Pi, configure a static IP:

```bash
# Example for Pi 1 (manager) at 192.168.1.10

sudo tee /etc/dhcpcd.conf >> /dev/null << 'EOF'
interface eth0
static ip_address=192.168.1.10/24
static routers=192.168.1.1
static domain_name_servers=8.8.8.8 1.1.1.1
EOF
```

Assign:
- Pi 1: `192.168.1.10` (Swarm Manager)
- Pi 2: `192.168.1.11` (Worker 1)
- Pi 3: `192.168.1.12` (Worker 2)

## Step 2: Install Docker on All Nodes

Run on each Pi:

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
sudo systemctl enable docker
```

## Step 3: Initialize Docker Swarm

On **Pi 1 (Manager)**:

```bash
# Initialize Swarm with the manager's IP
docker swarm init --advertise-addr 192.168.1.10

# Copy the worker join token from the output
# It looks like: docker swarm join --token SWMTKN-1-xxxxx 192.168.1.10:2377
```

## Step 4: Join Worker Nodes

On **Pi 2 and Pi 3**, run the join command from Step 3:

```bash
docker swarm join \
  --token SWMTKN-1-<token-from-step3> \
  192.168.1.10:2377
```

Verify on the manager:

```bash
docker node ls
# Should show all 3 nodes as Ready
```

## Step 5: Deploy Portainer on the Swarm

On the Swarm Manager, deploy Portainer as a Swarm stack:

```bash
curl -L https://downloads.portainer.io/ce2-19/portainer-agent-stack.yml \
  -o portainer-agent-stack.yml

docker stack deploy -c portainer-agent-stack.yml portainer
```

Or create the stack manually:

```bash
cat > portainer-stack.yml << 'EOF'
version: '3.2'

services:
  agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent_network
    deploy:
      mode: global    # Run agent on every Swarm node
      placement:
        constraints: [node.platform.os == linux]

  portainer:
    image: portainer/portainer-ce:latest
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - portainer_data:/data
    networks:
      - agent_network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]

networks:
  agent_network:
    driver: overlay
    attachable: true

volumes:
  portainer_data:
EOF

docker stack deploy -c portainer-stack.yml portainer
```

## Step 6: Access Portainer

Navigate to `http://192.168.1.10:9000` and create your admin account.

In Portainer you'll see:
- **Swarm** cluster overview
- All 3 nodes in the **Swarm > Nodes** section
- Ability to deploy **Services** and **Stacks** across the cluster

## Step 7: Deploy a Replicated Service

Test the cluster by deploying a replicated Nginx service:

```yaml
version: "3.8"

services:
  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    deploy:
      replicas: 3      # One replica per Pi
      update_config:
        parallelism: 1   # Update one replica at a time
        delay: 10s
      restart_policy:
        condition: on-failure
```

Portainer will distribute replicas across all 3 Pis automatically.

## Cluster Management with Portainer

### Draining a Node for Maintenance

1. In Portainer, navigate to **Swarm > Nodes**
2. Click on the node to maintenance
3. Set **Availability** to **Drain**
4. Portainer will migrate containers to other nodes

### Scaling Services

1. Navigate to **Services**
2. Click on a service
3. Change **Replicas** count and click **Update**

## Conclusion

A Raspberry Pi cluster running Docker Swarm with Portainer gives you a genuine highly available container platform for under $200. Portainer's Swarm support includes service deployment, node management, and rolling updates - all through a web interface. This setup is perfect for learning container orchestration before moving to production Kubernetes.
