# How to Track IPv4 Address Utilization and Prevent Exhaustion

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, IPAM, Monitoring, Address Management, Automation, NetBox

Description: Monitor IPv4 address utilization across subnets and implement alerting to prevent address exhaustion before it impacts operations.

IPv4 address exhaustion causes pod scheduling failures, VM provisioning errors, and service outages. Proactive monitoring prevents these incidents.

## Monitoring with NetBox API

```python
#!/usr/bin/env python3
# check_ip_utilization.py

import requests
import json

NETBOX_URL = "http://netbox.example.com"
TOKEN = "your-api-token"
ALERT_THRESHOLD = 80  # Alert when prefix is 80% utilized

headers = {
    "Authorization": f"Token {TOKEN}",
    "Content-Type": "application/json"
}

def check_prefix_utilization():
    """Check all active prefixes and alert on high utilization."""
    response = requests.get(
        f"{NETBOX_URL}/api/ipam/prefixes/",
        headers=headers,
        params={"status": "active", "limit": 1000}
    )
    prefixes = response.json()["results"]

    alerts = []
    for prefix in prefixes:
        if prefix.get("utilized") and prefix.get("mark_utilized") is False:
            utilized_pct = float(prefix.get("utilization", 0))
            if utilized_pct > ALERT_THRESHOLD:
                alerts.append({
                    "prefix": prefix["prefix"],
                    "description": prefix.get("description", ""),
                    "utilization": f"{utilized_pct:.1f}%",
                    "available": prefix.get("available", 0)
                })

    return alerts

alerts = check_prefix_utilization()
if alerts:
    print("HIGH UTILIZATION ALERT:")
    for a in alerts:
        print(f"  {a['prefix']} ({a['description']}): {a['utilization']} used, {a['available']} IPs available")
else:
    print("All prefixes within normal utilization")
```

## Monitoring Kubernetes IP Pool Utilization

```bash
#!/bin/bash
# check-calico-ipam.sh - Alert when Calico IP pool > 80% used

THRESHOLD=80

check_pool() {
    DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config \
        calicoctl ipam show --summary 2>/dev/null
}

# Parse output and check utilization
check_pool | awk '/IP Pool/{
    split($5, total, ",");
    split($6, used, ",");
    t = total[1]+0;
    u = used[1]+0;
    if (t > 0) {
        pct = int(u * 100 / t);
        print "Pool utilization: " u "/" t " = " pct "%";
        if (pct > '$THRESHOLD') {
            print "ALERT: IP pool utilization above '$THRESHOLD'%!";
            exit 1;
        }
    }
}'
```

## phpIPAM Subnet Utilization Report

```bash
# Get utilization for all subnets via phpIPAM API
curl -H "token: $TOKEN" \
  "http://phpipam.example.com/api/myapp/subnets/?filter_by=subnet&filter_value=10" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for subnet in data.get('data', []):
    used = int(subnet.get('used_hosts', 0))
    free = int(subnet.get('free_hosts', 0))
    total = used + free
    if total > 0:
        pct = used * 100 // total
        status = 'ALERT' if pct > 80 else 'OK'
        print(f'{status}: {subnet[\"subnet\"]}/{subnet[\"mask\"]} - {pct}% used ({used}/{total})')
"
```

## Prometheus Metrics for IP Utilization

```python
#!/usr/bin/env python3
# netbox_exporter.py - Export IP utilization metrics for Prometheus
from prometheus_client import start_http_server, Gauge
import requests, time

IPAM_UTILIZATION = Gauge(
    "ipam_prefix_utilization_percent",
    "IPv4 prefix utilization percentage",
    ["prefix", "description"]
)

def update_metrics():
    resp = requests.get(
        "http://netbox.example.com/api/ipam/prefixes/?limit=1000",
        headers={"Authorization": "Token your-token"}
    )
    for prefix in resp.json()["results"]:
        util = float(prefix.get("utilization", 0))
        IPAM_UTILIZATION.labels(
            prefix=prefix["prefix"],
            description=prefix.get("description", "")
        ).set(util)

if __name__ == "__main__":
    start_http_server(9100)
    while True:
        update_metrics()
        time.sleep(300)
```

## Alerting with Cron

```bash
# /etc/cron.d/ip-utilization-check
# Run every hour and email if any prefix is over 80%
0 * * * * root python3 /usr/local/bin/check_ip_utilization.py | \
  grep ALERT | mail -s "IP Utilization Alert" netops@example.com
```

Regular automated utilization monitoring prevents the "we ran out of IPs" incident that typically only gets discovered when a critical server fails to provision.
