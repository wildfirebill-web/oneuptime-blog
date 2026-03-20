# How to Monitor Docker Container IPv6 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Monitoring, Traffic Analysis, tcpdump, Metrics

Description: Monitor IPv6 traffic in Docker containers using tcpdump, netstat, and container stats, set up IPv6 traffic monitoring with Prometheus and cAdvisor, and analyze container IPv6 connection patterns.

## Introduction

Monitoring IPv6 traffic in Docker containers involves capturing traffic on bridge interfaces, analyzing per-container network statistics, and tracking IPv6 connection counts. Docker exposes network metrics through the stats API, and tools like tcpdump on bridge interfaces can capture all container IPv6 traffic. cAdvisor provides detailed per-container network metrics with Prometheus integration.

## Capture IPv6 Traffic on Docker Bridges

```bash
# Find bridge interface for a Docker network
BRIDGE=$(docker network inspect mynet \
    --format "{{.Options}}" | grep -o 'br-[a-f0-9]*' | head -1)

# If custom bridge name was set
BRIDGE="br-mynet"

# Capture all IPv6 traffic on the bridge
sudo tcpdump -i "$BRIDGE" -n ip6 -v

# Capture IPv6 HTTP traffic
sudo tcpdump -i "$BRIDGE" -n "ip6 and tcp port 80"

# Capture and save to file for analysis
sudo tcpdump -i "$BRIDGE" -n ip6 -w /tmp/docker-ipv6.pcap &
sleep 30
kill %1

# Analyze the capture
tcpdump -r /tmp/docker-ipv6.pcap -n ip6 | head -50
```

## Monitor Container Network Stats

```bash
# View real-time network stats for all containers
docker stats --format "table {{.Name}}\t{{.NetIO}}"

# Monitor specific container
docker stats mycontainer --no-stream \
    --format "{{.Name}}: NetIO={{.NetIO}}"

# Get detailed network stats via Docker API
curl -s --unix-socket /var/run/docker.sock \
    "http://localhost/containers/mycontainer/stats?stream=false" | \
    python3 -c "
import json, sys
stats = json.load(sys.stdin)
nets = stats.get('networks', {})
for iface, data in nets.items():
    print(f'Interface: {iface}')
    print(f'  RX bytes: {data[\"rx_bytes\"]}')
    print(f'  TX bytes: {data[\"tx_bytes\"]}')
    print(f'  RX packets: {data[\"rx_packets\"]}')
    print(f'  TX packets: {data[\"tx_packets\"]}')
"
```

## IPv6 Connections Inside Containers

```bash
# List IPv6 connections in a container
docker exec mycontainer sh -c "
    # Show all IPv6 TCP connections
    cat /proc/net/tcp6 | awk 'NR>1 {print \$3, \$4}' | head -20

    # Or use ss if available
    ss -t6 -n
"

# Count active IPv6 connections
docker exec mycontainer sh -c "
    ss -t6 | grep ESTAB | wc -l
"

# Monitor connection count over time
watch -n 5 'docker exec mycontainer ss -t6 | grep -c ESTAB'
```

## Prometheus Monitoring with cAdvisor

```yaml
# compose.yaml — IPv6 monitoring stack

networks:
  monitoring:
    driver: bridge
    enable_ipv6: true
    ipam:
      config:
        - subnet: 172.25.0.0/24
        - subnet: fd00:monitoring::/64

services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    ports:
      - "[::]:8080:8080"
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "[::]:9090:9090"
    networks:
      - monitoring
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

## Useful Prometheus Queries for Container IPv6

```promql
# Container network receive bytes per second (all interfaces)
rate(container_network_receive_bytes_total{name!=""}[5m])

# Container network transmit bytes (filter by container name)
rate(container_network_transmit_bytes_total{name="mycontainer"}[5m])

# Top containers by network traffic
topk(10, rate(container_network_receive_bytes_total{name!=""}[5m]))
```

## Conclusion

Monitor Docker IPv6 traffic using `tcpdump -i br-<id> ip6` to capture raw traffic on bridge interfaces, `docker stats` for aggregate network I/O, and the Docker stats API for per-container detailed metrics. Run cAdvisor with Prometheus for time-series container network monitoring with dashboards. Use `ss -t6` inside containers to monitor active IPv6 TCP connections. Bridge interface names follow the pattern `br-<first-12-chars-of-network-id>` — find them with `ip link show type bridge`.
