# How to Deploy Zabbix via Portainer for Infrastructure Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Zabbix, Monitoring, Infrastructure, Self-Hosted

Description: Deploy the full Zabbix monitoring stack including server, web frontend, and PostgreSQL database using Portainer stacks for comprehensive infrastructure monitoring.

## Introduction

Zabbix is one of the most powerful open-source infrastructure monitoring platforms, capable of monitoring servers, network devices, databases, applications, and cloud services. Deploying Zabbix via Portainer simplifies the multi-container setup with a single stack definition.

## Prerequisites

- Docker host with at least 4GB RAM and 2 CPUs
- Portainer CE or BE installed
- Ports 8080 (web UI) and 10051 (Zabbix trapper) available

## Deploying the Zabbix Stack

### Step 1: Create the Stack

Navigate to **Stacks > Add stack** in Portainer. Name it `zabbix` and paste:

```yaml
version: "3.8"

services:
  # PostgreSQL database for Zabbix
  zabbix-db:
    image: postgres:15-alpine
    container_name: zabbix-db
    environment:
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_secure_password
    volumes:
      - zabbix-db-data:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U zabbix"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Zabbix Server - core monitoring process
  zabbix-server:
    image: zabbix/zabbix-server-pgsql:alpine-7.0-latest
    container_name: zabbix-server
    environment:
      DB_SERVER_HOST: zabbix-db
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_secure_password
      ZBX_JAVAGATEWAY_ENABLE: "true"
      ZBX_STARTJAVAPOLLERS: 5
    ports:
      - "10051:10051"  # Zabbix trapper port for agents
    volumes:
      - zabbix-server-data:/var/lib/zabbix
      - zabbix-snmptraps:/var/lib/zabbix/snmptraps
    depends_on:
      zabbix-db:
        condition: service_healthy
    restart: unless-stopped

  # Zabbix Web Frontend (Nginx + PHP-FPM)
  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest
    container_name: zabbix-web
    environment:
      ZBX_SERVER_HOST: zabbix-server
      DB_SERVER_HOST: zabbix-db
      POSTGRES_DB: zabbix
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix_secure_password
      PHP_TZ: America/New_York
    ports:
      - "8080:8080"   # HTTP
      - "8443:8443"   # HTTPS
    depends_on:
      - zabbix-server
    restart: unless-stopped

  # Zabbix Agent2 - monitors the Docker host itself
  zabbix-agent:
    image: zabbix/zabbix-agent2:alpine-7.0-latest
    container_name: zabbix-agent
    environment:
      ZBX_HOSTNAME: docker-host
      ZBX_SERVER_HOST: zabbix-server
      ZBX_SERVER_PORT: 10051
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc:/proc:ro
      - /sys:/sys:ro
    privileged: true
    restart: unless-stopped

  # Java Gateway for JMX monitoring
  zabbix-java-gateway:
    image: zabbix/zabbix-java-gateway:alpine-7.0-latest
    container_name: zabbix-java-gateway
    restart: unless-stopped

volumes:
  zabbix-db-data:
    driver: local
  zabbix-server-data:
    driver: local
  zabbix-snmptraps:
    driver: local
```

### Step 2: Deploy the Stack

Click **Deploy the stack**. The initial deployment takes 2–3 minutes for the database to initialize and Zabbix Server to apply the schema.

Monitor progress in **Containers** - wait until all five containers show as **running**.

### Step 3: Access the Web Interface

Open `http://<your-host>:8080` in a browser.

Default credentials:
- Username: `Admin`
- Password: `zabbix`

**Change the password immediately** via **User Settings > Change password**.

### Step 4: Add Hosts for Monitoring

1. Navigate to **Data Collection > Hosts**
2. Click **Create host**
3. Set **Host name** and **Interfaces** (add Agent interface with the target IP)
4. Link a **Template** such as `Linux by Zabbix agent` or `Windows by Zabbix agent`
5. Click **Add**

### Step 5: Install Zabbix Agent on Remote Hosts

For Linux targets:

```bash
# Install Zabbix agent on Ubuntu/Debian

wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-1+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_7.0-1+ubuntu22.04_all.deb
sudo apt update && sudo apt install -y zabbix-agent2

# Configure the agent
sudo tee /etc/zabbix/zabbix_agent2.conf > /dev/null << EOF
Server=<your-docker-host-ip>
ServerActive=<your-docker-host-ip>
Hostname=$(hostname)
EOF

sudo systemctl enable --now zabbix-agent2
```

### Step 6: Configure Email Alerts

1. Navigate to **Alerts > Media types**
2. Click **Email** and configure your SMTP server
3. Navigate to **Users > Admin > Media**
4. Add an email media entry
5. Create an **Action** under **Alerts > Actions > Trigger actions**

## Scaling Considerations

For production use, consider:

```yaml
# Add to zabbix-server environment for performance tuning
ZBX_STARTPOLLERS: 10
ZBX_STARTPINGERS: 5
ZBX_STARTDISCOVERERS: 3
ZBX_CACHESIZE: 256M
ZBX_HISTORYCACHESIZE: 64M
```

## Conclusion

Zabbix deployed via Portainer provides enterprise-grade infrastructure monitoring with a familiar web interface. The stack approach makes it easy to version-control your configuration and redeploy consistently. With Zabbix Agent2 installed on your hosts, you get deep visibility into CPU, memory, disk, network, and application-level metrics.
