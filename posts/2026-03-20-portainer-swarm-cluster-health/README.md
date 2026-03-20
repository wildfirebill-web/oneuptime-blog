# How to Monitor Docker Swarm Cluster Health from Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Monitoring, Health, Infrastructure

Description: Monitor Docker Swarm cluster health including node status, service availability, and task failures using Portainer and complementary tools.

## Introduction

A healthy Docker Swarm cluster requires monitoring at multiple levels: node health, service availability, task states, and resource utilization. Portainer provides built-in Swarm visibility, and combined with Prometheus and Grafana, you can build comprehensive cluster health monitoring.

## Portainer Built-In Swarm Monitoring

Portainer provides several views for Swarm health:

1. **Home > Swarm Environment**: Overall cluster status, node count
2. **Swarm > Nodes**: Individual node health and resource usage
3. **Swarm > Services**: Service task distribution and health
4. **Swarm > Tasks**: Individual task states across the cluster

## Health Check Script

```bash
#!/bin/bash
# swarm-health-check.sh

PORTAINER_URL="https://portainer.example.com"
API_KEY="your-api-key"
ENDPOINT_ID=1

echo "=== Docker Swarm Health Report ==="
echo "Time: $(date)"
echo ""

# Node health
echo "--- Nodes ---"
curl -s \
  -H "X-API-Key: $API_KEY" \
  "$PORTAINER_URL/api/endpoints/$ENDPOINT_ID/docker/nodes" \
  | python3 -c "
import sys, json
nodes = json.load(sys.stdin)
for n in nodes:
    hostname = n['Description']['Hostname']
    state = n['Status']['State']
    avail = n['Spec']['Availability']
    role = n['Spec']['Role']
    manager_status = n.get('ManagerStatus', {}).get('Reachability', 'N/A')
    
    status_icon = '✓' if state == 'ready' else '✗'
    print(f'{status_icon} {hostname:20} {role:8} {state:10} {avail:8} {manager_status}')
"

echo ""
echo "--- Services ---"
curl -s \
  -H "X-API-Key: $API_KEY" \
  "$PORTAINER_URL/api/endpoints/$ENDPOINT_ID/docker/services" \
  | python3 -c "
import sys, json
services = json.load(sys.stdin)
for s in services:
    name = s['Spec']['Name']
    mode = s['Spec']['Mode']
    if 'Replicated' in mode:
        desired = mode['Replicated'].get('Replicas', 0)
        running = s.get('ServiceStatus', {}).get('RunningTasks', 0)
        status_icon = '✓' if running >= desired else '!'
        print(f'{status_icon} {name:30} {running}/{desired} replicas')
    else:
        print(f'✓ {name:30} global')
"

echo ""
echo "--- Failed Tasks (last 10 minutes) ---"
curl -s \
  -H "X-API-Key: $API_KEY" \
  "$PORTAINER_URL/api/endpoints/$ENDPOINT_ID/docker/tasks" \
  | python3 -c "
import sys, json
from datetime import datetime, timezone, timedelta
tasks = json.load(sys.stdin)
cutoff = datetime.now(timezone.utc) - timedelta(minutes=10)
failed = [t for t in tasks 
          if t['Status']['State'] in ('failed', 'rejected', 'orphaned')]
if failed:
    for t in failed[-10:]:
        svc = t.get('ServiceID', 'N/A')[:12]
        state = t['Status']['State']
        err = t['Status'].get('Err', 'no error')[:50]
        print(f'  ✗ {svc}: {state} - {err}')
else:
    print('  No failed tasks')
"
```

## Prometheus Stack for Swarm Monitoring

```yaml
# monitoring-stack.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
    ports:
      - "9090:9090"
    volumes:
      - prometheus-config:/etc/prometheus
      - prometheus-data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=30d
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    deploy:
      mode: global       # Run on every node
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    deploy:
      mode: global       # Container metrics from every node
    volumes:
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    deploy:
      replicas: 1
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    networks:
      - monitoring

volumes:
  prometheus-config:
  prometheus-data:
  grafana-data:

networks:
  monitoring:
    driver: overlay
    attachable: true
```

## Key Metrics to Monitor

```yaml
# prometheus.yml rules for Swarm health
groups:
  - name: swarm_health
    rules:
      # Node down
      - alert: SwarmNodeDown
        expr: up{job="node-exporter"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Swarm node {{ $labels.instance }} is down"

      # Service under-replicated
      - alert: SwarmServiceUnderReplicated
        expr: docker_swarm_service_running_tasks < docker_swarm_service_desired_tasks
        for: 5m
        labels:
          severity: warning

      # High CPU on a node
      - alert: SwarmNodeHighCPU
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
```

## Conclusion

Comprehensive Swarm cluster health monitoring combines Portainer's built-in visibility with Prometheus metrics and Grafana dashboards. The multi-level monitoring approach covers node health, service availability, task failures, and resource utilization. Automated alerting ensures operations teams are notified immediately when cluster health degrades, enabling rapid response.
