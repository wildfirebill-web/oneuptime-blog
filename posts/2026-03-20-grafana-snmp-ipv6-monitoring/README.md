# How to Monitor IPv6 with SNMP using Grafana

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Grafana, SNMP, IPv6, Monitoring, Network, MIB

Description: A guide to monitoring IPv6 statistics from network devices using SNMP, the SNMP Exporter for Prometheus, and Grafana dashboards.

SNMP (Simple Network Management Protocol) provides IPv6 statistics through RFC 4293 (IP-MIB) and RFC 4022 (TCP-MIB). This guide covers scraping IPv6 SNMP metrics from network devices and visualizing them in Grafana.

## Step 1: Install and Configure SNMP Exporter

```bash
# Download SNMP exporter

wget https://github.com/prometheus/snmp_exporter/releases/latest/download/snmp_exporter-*.linux-amd64.tar.gz
tar xzf snmp_exporter-*.linux-amd64.tar.gz
sudo mv snmp_exporter /usr/local/bin/
```

## Step 2: Generate SNMP Module for IPv6 MIBs

Use the `generator` tool to create a config for IPv6-related OIDs:

```yaml
# generator.yml - Generate SNMP config for IPv6 statistics
modules:
  ipv6_stats:
    walk:
      # IP-MIB IPv6 statistics (RFC 4293)
      - 1.3.6.1.2.1.4.24    # ipv6RouteTable
      - 1.3.6.1.2.1.55      # ipv6MIB
      - 1.3.6.1.2.1.56      # ipv6IfTable
    metrics:
      - name: ipv6IfStatInReceives
        oid: 1.3.6.1.2.1.56.1.1.1.6
        type: counter
        help: "Total IPv6 datagrams received on the interface"
      - name: ipv6IfStatOutRequests
        oid: 1.3.6.1.2.1.56.1.1.1.22
        type: counter
        help: "Total IPv6 datagrams sent from the interface"
```

```bash
# Generate snmp.yml from generator.yml
snmp_exporter/generator generate
```

## Step 3: Configure snmp.yml for IPv6 Devices

```yaml
# snmp.yml - SNMP module for IPv6 network devices
modules:
  cisco_ios_ipv6:
    walk:
      - sysUpTime
      - ifXTable
      - ipv6IfTable
      - ipv6IfStatsTable
    auth:
      community: "{{ snmp_community }}"
    version: 2
    timeout: 15s
    retries: 3

  linux_ipv6:
    walk:
      - ipv6IfTable
      - ipv6IfStatsTable
    auth:
      community: public
    version: 2
```

## Step 4: Prometheus Configuration for SNMP + IPv6

```yaml
# prometheus.yml - Scrape IPv6 stats via SNMP Exporter
scrape_configs:
  # Scrape IPv6 stats from network devices
  - job_name: "snmp-ipv6-devices"
    static_configs:
      - targets:
          - "2001:db8::router1"   # IPv6 address of the device
          - "2001:db8::switch1"
    metrics_path: /snmp
    params:
      module: [cisco_ios_ipv6]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        # SNMP Exporter listening on IPv6
        replacement: "[::1]:9116"
```

## Step 5: Start SNMP Exporter on IPv6

```bash
# Start SNMP Exporter listening on IPv6
snmp_exporter --config.file=snmp.yml \
  --web.listen-address="[::]:9116"
```

## Step 6: Grafana Dashboard for SNMP IPv6 Metrics

Useful PromQL queries for Grafana panels:

```promql
# IPv6 interface receive rate (packets/sec)
rate(ipv6IfStatInReceives{instance=~"$device"}[5m])

# IPv6 interface transmit rate
rate(ipv6IfStatOutRequests{instance=~"$device"}[5m])

# IPv6 routing table size
ipv6RouteNumber

# IPv6 interface errors
rate(ipv6IfStatInDiscards{instance=~"$device"}[5m])
rate(ipv6IfStatOutDiscards{instance=~"$device"}[5m])
```

## Step 7: Import Community SNMP IPv6 Dashboard

```bash
# Import the SNMP Statistics dashboard from Grafana.com (dashboard ID varies)
# Or use the Grafana UI: Dashboards > Import > Enter ID
curl -s "https://grafana.com/api/dashboards/11169/revisions/1/download" \
  | curl -X POST http://admin:admin@localhost:3000/api/dashboards/import \
    -H "Content-Type: application/json" \
    --data-binary @-
```

## Verify SNMP IPv6 Collection

```bash
# Test manual SNMP query to IPv6 device
snmpwalk -v2c -c public "[2001:db8::router1]" ipv6IfTable

# Test SNMP Exporter is collecting IPv6 stats
curl "http://[::1]:9116/snmp?module=cisco_ios_ipv6&target=2001:db8::router1" | \
  grep ipv6
```

Combining SNMP with the SNMP Exporter and Grafana provides comprehensive IPv6 visibility for network devices that may not have modern telemetry APIs, making it essential for monitoring legacy hardware alongside modern cloud infrastructure.
