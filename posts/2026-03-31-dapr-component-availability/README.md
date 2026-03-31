# How to Monitor Dapr Component Availability

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Component, Monitoring, Availability, Prometheus

Description: Monitor Dapr component availability to detect when state stores, pub/sub brokers, and bindings become unreachable before they impact your application workloads.

---

## What Is Component Availability?

Dapr components (state stores, pub/sub brokers, secret stores, bindings) connect to external infrastructure like Redis, Kafka, and databases. When these external systems become unavailable, the corresponding Dapr component becomes degraded. Monitoring component availability helps you detect infrastructure problems early.

## Checking Component Status via API

The Dapr metadata endpoint reports the health status of each configured component:

```bash
# Query component status
curl http://localhost:3500/v1.0/metadata | python3 -m json.tool

# Check specifically for unhealthy components
curl -s http://localhost:3500/v1.0/metadata | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
for c in data.get('components', []):
    status = c.get('status', 'UNKNOWN')
    print(f\"{c['name']} ({c['type']}): {status}\")
"
```

## Using the Outbound Health Endpoint

A simpler approach is the outbound health endpoint - it returns 204 only when all components are healthy:

```bash
# Returns 204 if all components healthy, 500 otherwise
curl -o /dev/null -w "%{http_code}" \
  http://localhost:3500/v1.0/healthz/outbound
```

## Prometheus Metrics for Component Health

Track component operation success rates:

```bash
# State store availability (success rate)
rate(dapr_component_state_count{success="true"}[5m])
/
rate(dapr_component_state_count[5m])

# Pub/sub availability
rate(dapr_component_pubsub_publish_count{success="true"}[5m])
/
rate(dapr_component_pubsub_publish_count[5m])

# Binding availability
rate(dapr_component_binding_count{success="true"}[5m])
/
rate(dapr_component_binding_count[5m])
```

## Alerting on Component Degradation

```yaml
groups:
  - name: component-availability
    rules:
      - alert: DaprStateStoreDegraded
        expr: >
          rate(dapr_component_state_count{success="false"}[5m])
          /
          rate(dapr_component_state_count[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Dapr state store error rate exceeds 10% for component {{ $labels.component }}"

      - alert: DaprPubSubDegraded
        expr: >
          rate(dapr_component_pubsub_publish_count{success="false"}[3m]) > 1
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "Dapr pub/sub component {{ $labels.component }} has persistent failures"
```

## Implementing Application-Level Component Checks

Periodically probe component availability from your application:

```python
import requests
import logging

logger = logging.getLogger(__name__)

def check_component_availability():
    components = {
        "statestore": "state/statestore/health",
        "pubsub": "metadata",  # Use metadata as proxy
    }

    for name, path in components.items():
        try:
            response = requests.get(
                f"http://localhost:3500/v1.0/{path}",
                timeout=2
            )
            status = "OK" if response.ok else "DEGRADED"
        except Exception as e:
            status = "UNREACHABLE"

        logger.info(f"Component {name}: {status}")
        record_metric(f"dapr_component_{name}_available", status == "OK")
```

## Summary

Monitor Dapr component availability using the metadata API for detailed status, the outbound health endpoint for a binary healthy/unhealthy signal, and Prometheus metrics for success rate trends. Set up alerts for sustained component failure rates and implement application-level component probes to get early warning of infrastructure degradation before it triggers cascading failures.
