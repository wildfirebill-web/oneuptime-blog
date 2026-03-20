# How to Manage Multiple Game Servers with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Game Server, Portainer, Docker, Multi-Server, Gaming, Self-Hosted, Management

Description: Use Portainer to manage a fleet of game servers from a single dashboard, with shared infrastructure, resource limits, and centralized monitoring for Minecraft, Valheim, Rust, and more.

---

Running multiple game servers on a single host requires careful resource management and organization. Portainer's stacks, resource limits, and monitoring features make it practical to host multiple games on one server without them interfering with each other.

## Architecture for Multi-Game Hosting

```mermaid
graph TB
    Portainer[Portainer Dashboard] --> Stack1[Minecraft Stack]
    Portainer --> Stack2[Valheim Stack]
    Portainer --> Stack3[Rust Stack]
    Portainer --> Stack4[Factorio Stack]
    Stack1 --> Network1[game-network-1]
    Stack2 --> Network2[game-network-2]
    Stack3 --> Network3[game-network-3]
    Stack4 --> Network4[game-network-4]
```

## Step 1: Resource Planning

Before deploying, plan resource allocation:

| Game | RAM | CPU | Disk | Players |
|------|-----|-----|------|---------|
| Minecraft (Paper) | 4GB | 2 cores | 5GB | 20 |
| Valheim | 4GB | 2 cores | 2GB | 10 |
| Rust | 8GB | 4 cores | 20GB | 50 |
| Factorio | 2GB | 1 core | 5GB | 10 |
| **Total** | **18GB** | **9 cores** | **32GB** | - |

## Step 2: Deploy with Resource Limits

Always set resource limits to prevent one server from starving others:

```yaml
# minecraft-with-limits.yml

version: "3.8"

services:
  minecraft:
    image: itzg/minecraft-server:latest
    environment:
      - EULA=TRUE
      - MEMORY=3G
    deploy:
      resources:
        limits:
          memory: 4G       # Hard memory limit
          cpus: "2.0"      # Max 2 CPU cores
        reservations:
          memory: 2G       # Guaranteed minimum
    volumes:
      - minecraft-data:/data
    ports:
      - "25565:25565"
    restart: unless-stopped
    networks:
      - minecraft-net

networks:
  minecraft-net:
    driver: bridge

volumes:
  minecraft-data:
```

Apply similar resource limits to each game server stack.

## Step 3: Centralized Reverse Proxy

Add a reverse proxy for web management interfaces:

```yaml
# nginx-proxy-stack.yml
version: "3.8"

services:
  nginx:
    image: nginx:1.25-alpine
    volumes:
      - /opt/nginx/conf.d:/etc/nginx/conf.d:ro
    ports:
      - "80:80"
      - "443:443"
    restart: unless-stopped

  # Auto-configure Nginx from container labels
  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    volumes:
      - npm-data:/data
      - npm-letsencrypt:/etc/letsencrypt
    ports:
      - "81:81"    # NPM admin interface
    restart: unless-stopped
```

## Step 4: Shared Monitoring Stack

Deploy a single monitoring stack that covers all game servers:

```yaml
# monitoring-stack.yml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:v2.50.0
    volumes:
      - /opt/monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:10.3.0
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

  node-exporter:
    image: prom/node-exporter:latest
    pid: host
    network_mode: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
```

## Step 5: Schedule Restarts and Maintenance

Use Portainer's scheduled jobs (or Watchtower) to restart game servers during off-peak hours:

```bash
#!/bin/bash
# restart-all-servers.sh
# Run via Portainer scheduled job at 4 AM server time

SERVERS=("minecraft_minecraft_1" "valheim_valheim_1" "factorio_factorio_1")

for server in "${SERVERS[@]}"; do
    echo "Restarting $server..."
    docker restart "$server"
    sleep 30    # Wait for server to come up before restarting the next
done
```

## Tips for Multi-Game Hosting

- **Separate stacks per game** - keeps resource limits and networks isolated
- **Use named volumes** - makes backup scripts simpler
- **Label all containers** - helps filter by game in Portainer's container list
- **Monitor disk usage** - game servers accumulate large world files
- **Schedule wipes/backups off-peak** - avoids impacting active players

## Summary

Portainer makes managing a fleet of game servers practical. Resource limits prevent any single server from monopolizing the host, separate stacks provide isolation, and the centralized dashboard gives you visibility across all servers from one place.
