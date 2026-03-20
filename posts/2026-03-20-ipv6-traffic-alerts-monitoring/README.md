# How to Create IPv6 Traffic Alerts in Monitoring Systems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Alerting, Prometheus, Grafana, Monitoring, Network Operations

Description: A guide to creating meaningful IPv6 traffic alerts in Prometheus and Grafana that detect anomalies, failures, and capacity issues.

Effective IPv6 alerting requires monitoring for both failures (endpoint down, routing issues) and anomalies (traffic spikes, sudden drops, latency increases). This guide covers practical alert rules for IPv6 environments.

## Alert Category 1: IPv6 Endpoint Availability

```yaml
# alerts-ipv6-availability.yml
groups:
  - name: ipv6-availability
    rules:
      # HTTP endpoint down over IPv6
      - alert: IPv6HTTPDown
        expr: probe_success{job="blackbox-http-ipv6"} == 0
        for: 2m
        labels:
          severity: critical
          category: availability
        annotations:
          summary: "IPv6 HTTP endpoint {{ $labels.instance }} is DOWN"
          runbook: "https://wiki.example.com/runbooks/ipv6-endpoint-down"

      # ICMP unreachable over IPv6
      - alert: IPv6PingFailing
        expr: probe_success{job="blackbox-icmp-ipv6"} == 0
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "IPv6 ICMP to {{ $labels.instance }} is failing"
```

## Alert Category 2: IPv6 Routing Issues

```yaml
      # No-route errors indicate IPv6 routing failures
      - alert: IPv6NoRouteErrors
        expr: rate(node_netstat_Ip6_OutNoRoutes[5m]) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "IPv6 routing failures on {{ $labels.instance }}"

      # IPv6 fragmentation needed (PMTUD issues)
      - alert: IPv6PMTUDFailures
        expr: rate(node_netstat_Ip6_FragFails[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "IPv6 fragmentation failures on {{ $labels.instance }}"
```

## Alert Category 3: IPv6 Traffic Anomalies

```yaml
      # Sudden drop in IPv6 traffic (could indicate routing issue)
      - alert: IPv6TrafficDrop
        expr: >
          (
            rate(node_netstat_Ip6_InReceives[5m]) <
            rate(node_netstat_Ip6_InReceives[1h] offset 1h) * 0.1
          )
          AND
          rate(node_netstat_Ip6_InReceives[1h] offset 1h) > 100
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "IPv6 traffic dropped significantly on {{ $labels.instance }}"

      # IPv6 traffic spike (possible DDoS or misconfiguration)
      - alert: IPv6TrafficSpike
        expr: >
          rate(node_netstat_Ip6_InReceives[5m]) >
          rate(node_netstat_Ip6_InReceives[1h] offset 1h) * 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "IPv6 traffic spike on {{ $labels.instance }}"
```

## Alert Category 4: IPv6 BGP Session Alerts

```yaml
      # BGP session not established
      - alert: IPv6BGPSessionDown
        expr: bgp_session_state{peer=~".*:.*"} != 6
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "IPv6 BGP session to {{ $labels.peer }} is DOWN (state={{ $value }})"

      # BGP prefix count dropped significantly
      - alert: IPv6BGPPrefixCountDrop
        expr: >
          bgp_prefixes_received_count{afi="ipv6"} <
          bgp_prefixes_received_count{afi="ipv6"} offset 30m * 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "IPv6 BGP prefix count dropped on {{ $labels.peer }}"
```

## Alert Category 5: IPv6 Adoption Regression

```yaml
      # IPv6 adoption drops (could indicate dual-stack misconfiguration)
      - alert: IPv6AdoptionRegression
        expr: >
          sum(probe_success{job="blackbox-http-ipv6"}) /
          sum(probe_success{job="blackbox-http-ipv4"}) < 0.95
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "IPv6 endpoint availability is lower than IPv4 (possible regression)"
```

## Grafana Alert Rules (UI)

In Grafana Alerting → Alert Rules → Create:

```
Alert name: IPv6 Endpoint Latency High
Condition: avg over last 5m of probe_duration_seconds{job="blackbox-http-ipv6"} > 2
For: 5 minutes
Labels: severity=warning, category=latency
Notifications: ops-slack-channel
```

## Alert Delivery Configuration

```yaml
# alertmanager.yml - Route IPv6 alerts to appropriate channels
route:
  group_by: ['alertname', 'instance']
  routes:
    - match:
        category: availability
        severity: critical
      receiver: pagerduty-critical
    - match:
        category: routing
      receiver: netops-slack
    - match_re:
        alertname: "IPv6.*"
      receiver: ipv6-team-slack
```

Well-structured IPv6 traffic alerts with clear severity levels and runbook links enable rapid incident response for IPv6-specific issues before they affect a significant portion of users.
