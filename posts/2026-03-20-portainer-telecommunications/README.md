# How to Use Portainer in Telecommunications Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Telecommunications, NFV, 5G, Network Functions

Description: Deploy and manage Network Function Virtualization (NFV) and telecom-grade containerized workloads using Portainer for multi-site telecommunications infrastructure.

## Introduction

Telecommunications providers are virtualizing network functions through NFV (Network Function Virtualization) and containerizing network services as CNFs (Containerized Network Functions). From VoIP gateways to 5G core components, containers are transforming telecom infrastructure. Portainer provides a unified management plane for containerized telecom workloads across central data centers and distributed edge Points of Presence (PoPs).

## Telecom Container Use Cases

- VoIP gateways and SBC (Session Border Controllers)
- DNS resolvers and routing services
- Network monitoring and OSS/BSS systems
- 5G core network functions (AMF, SMF, UPF)
- Billing and mediation systems
- Network analytics and traffic monitoring

## Step 1: High-Performance Docker Configuration for Telecom

```bash
# Telecom workloads require specific kernel parameters
cat >> /etc/sysctl.conf << 'EOF'
# Network performance tuning for telecom
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 65536 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
net.core.netdev_max_backlog = 300000
net.ipv4.tcp_low_latency = 1
net.ipv4.udp_rmem_min = 65536
net.ipv4.udp_wmem_min = 65536
EOF

sysctl -p

# Docker daemon optimized for high-throughput networking
cat > /etc/docker/daemon.json << 'EOF'
{
  "userland-proxy": false,
  "live-restore": true,
  "icc": false,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  },
  "mtu": 1500
}
EOF
```

## Step 2: Deploy VoIP Infrastructure

```yaml
# voip-stack/docker-compose.yml
version: '3.8'
services:
  kamailio:
    image: telecom/kamailio:5.7
    restart: always
    network_mode: host    # Required for RTP media handling
    environment:
      - DOMAIN=sip.telecom.com
      - DB_URL=mysql://kamailio:password@db/kamailio
      - RTP_ENGINE_HOST=rtpengine
    volumes:
      - ./kamailio.cfg:/etc/kamailio/kamailio.cfg:ro
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 2g

  rtpengine:
    image: telecom/rtpengine:mr11.5
    restart: always
    network_mode: host
    cap_add:
      - NET_ADMIN   # Required for traffic control
    environment:
      - INTERFACE=eth0
      - PORT_RANGE=30000-40000
    command: --interface=eth0 --listen-ng=127.0.0.1:2223 --port-min=30000 --port-max=40000

  asterisk:
    image: telecom/asterisk:20-alpine
    restart: always
    ports:
      - "5060:5060/udp"
      - "5060:5060/tcp"
      - "10000-10100:10000-10100/udp"   # RTP range
    volumes:
      - ./asterisk/:/etc/asterisk/:ro
      - asterisk-spool:/var/spool/asterisk
    environment:
      - ASTERISK_REALM=telecom.com

  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: kamailio
      MYSQL_USER: kamailio
      MYSQL_PASSWORD: password
    volumes:
      - voip-db:/var/lib/mysql

volumes:
  asterisk-spool:
  voip-db:
```

## Step 3: DNS Infrastructure with Anycast

```yaml
# dns-stack/docker-compose.yml
version: '3.8'
services:
  pdns-authoritative:
    image: powerdns/pdns-auth-48:latest
    restart: always
    ports:
      - "53:53/udp"
      - "53:53/tcp"
    environment:
      - PDNS_launch=gmysql
      - PDNS_gmysql_host=dns-db
      - PDNS_gmysql_user=pdns
      - PDNS_gmysql_password=pdnspass
      - PDNS_gmysql_dbname=pdns
    deploy:
      replicas: 2
      placement:
        preferences:
          - spread: node.labels.datacenter  # Spread across DCs

  pdns-recursor:
    image: powerdns/pdns-recursor-49:latest
    restart: always
    ports:
      - "5353:53/udp"   # Resolver port
    volumes:
      - ./recursor.conf:/etc/pdns-recursor/recursor.conf:ro
    deploy:
      replicas: 3

  dns-db:
    image: mysql:8.0
    restart: always
    volumes:
      - dns-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: pdns
      MYSQL_USER: pdns
      MYSQL_PASSWORD: pdnspass

volumes:
  dns-data:
```

## Step 4: Network Monitoring Stack

```yaml
# network-monitoring/docker-compose.yml
version: '3.8'
services:
  pmacct:
    image: telecom/pmacct:1.7.9
    restart: always
    network_mode: host
    volumes:
      - ./pmacctd.conf:/etc/pmacct/pmacctd.conf:ro
    cap_add:
      - NET_ADMIN

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    restart: always
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    restart: always
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  flow-analyzer:
    image: telecom/flow-analyzer:v2.3
    restart: always
    environment:
      - KAFKA_BROKERS=kafka:9092
      - INFLUX_URL=http://influxdb:8086
      - ALERT_THRESHOLD_GBPS=10.0
    depends_on:
      - kafka
```

## Step 5: Multi-PoP Deployment with Portainer Edge

```bash
# Register each PoP as an Edge environment
for pop in nyc-pop-1 lax-pop-1 chi-pop-1 mia-pop-1; do
  echo "Setting up $pop..."
  
  # On each PoP server, deploy edge agent
  docker run -d \
    --name portainer-agent \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v agent-data:/data \
    -e EDGE=1 \
    -e EDGE_ID="$pop" \
    -e EDGE_KEY="$PORTAINER_EDGE_KEY" \
    -e EDGE_SERVER_HOST="https://portainer.telecom.com:9443" \
    portainer/agent:latest
done

# Deploy telecom services to all PoPs simultaneously
# In Portainer: Edge Stacks > Add Edge Stack > Select "All PoPs" group
```

## Step 6: SLA Monitoring and Alerting

```bash
#!/bin/bash
# telecom-sla-monitor.sh
PORTAINER_URL="https://portainer.telecom.com"
API_KEY="noc-api-key"
PAGERDUTY_KEY="pd-integration-key"

# Check critical telecom services
SERVICES=("kamailio" "rtpengine" "pdns-authoritative" "pdns-recursor")

for service in "${SERVICES[@]}"; do
  REPLICAS=$(curl -s \
    -H "X-API-Key: $API_KEY" \
    "$PORTAINER_URL/api/endpoints/1/docker/services/$service" | \
    python3 -c "
import sys,json
try:
    s = json.load(sys.stdin)
    desired = s['Spec']['Mode']['Replicated']['Replicas']
    running = s['ServiceStatus']['RunningTasks']
    print(f'{running}/{desired}')
except:
    print('0/0')
")

  RUNNING=$(echo $REPLICAS | cut -d'/' -f1)
  DESIRED=$(echo $REPLICAS | cut -d'/' -f2)

  if [ "$RUNNING" != "$DESIRED" ]; then
    # Trigger PagerDuty alert
    curl -s -X POST \
      -H "Content-Type: application/json" \
      -d "{
        \"routing_key\": \"$PAGERDUTY_KEY\",
        \"event_action\": \"trigger\",
        \"payload\": {
          \"summary\": \"Telecom Service Degraded: $service ($REPLICAS)\",
          \"severity\": \"critical\",
          \"source\": \"portainer-monitor\"
        }
      }" \
      "https://events.pagerduty.com/v2/enqueue"
    echo "ALERT: $service is degraded ($REPLICAS)"
  else
    echo "OK: $service ($REPLICAS)"
  fi
done
```

## Conclusion

Telecommunications containerized workloads require careful network configuration including host networking for RTP media, kernel parameter tuning for high-throughput traffic, and multi-site deployment capabilities. Portainer's Edge Stack feature enables simultaneous deployment of VoIP, DNS, and monitoring services across distributed Points of Presence from a single control plane. Combined with SLA monitoring scripts and PagerDuty alerting, Portainer provides the operational visibility that NOC teams need to maintain carrier-grade service levels.
