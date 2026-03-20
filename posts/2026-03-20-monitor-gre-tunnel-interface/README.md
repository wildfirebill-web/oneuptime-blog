# How to Monitor GRE Tunnel Interface Status

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GRE, Tunnel, Monitoring, Linux, IPv4, ip monitor, Prometheus, Alerting

Description: Learn how to monitor GRE tunnel interface status on Linux including link state, traffic counters, packet loss, and how to set up alerts using standard tools and Prometheus.

---

Monitoring GRE tunnel health involves checking interface state, traffic flow, packet loss, and tunnel endpoint reachability to detect failures before they impact applications.

## Basic Status Checks

```bash
# Check interface operational state
ip link show gre1

# Output:
# gre1: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1476 qdisc noqueue state UNKNOWN
#   link/gre 10.0.0.1 peer 10.0.0.2

# State UNKNOWN is normal for GRE tunnels (not connected interfaces)
# The tunnel is UP if LOWER_UP flag is present

# Check tunnel IP and routing
ip addr show gre1
ip route show dev gre1

# Ping through the tunnel
ping -c5 172.16.1.2   # Inner IP of remote tunnel endpoint
```

## Traffic Counters

```bash
# Show TX/RX bytes and packet counts
ip -s link show gre1

# Output:
# RX: bytes  packets  errors  dropped  overrun  mcast
#     123456      100       0        0        0      0
# TX: bytes  packets  errors  dropped carrier collsns
#     654321      200       0        0       0       0

# Watch counters in real time
watch -n 2 "ip -s link show gre1"
```

## Monitoring with ip monitor

```bash
# Watch for tunnel interface state changes
ip monitor link | grep gre1

# Redirect to log file
ip monitor link 2>&1 | while read line; do
  if echo "$line" | grep -q "gre1"; then
    echo "$(date): $line" >> /var/log/gre-monitor.log
  fi
done &
```

## Prometheus: node_exporter Metrics

```bash
# node_exporter exposes interface statistics
# Useful metrics:
# node_network_up{device="gre1"} 1.0           (1=up, 0=down)
# node_network_receive_bytes_total{device="gre1"}
# node_network_transmit_bytes_total{device="gre1"}
# node_network_receive_drop_total{device="gre1"}

# Prometheus alert rule
```

```yaml
groups:
  - name: gre_tunnel
    rules:
      - alert: GRETunnelDown
        expr: node_network_up{device="gre1"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "GRE tunnel gre1 is down on {{ $labels.instance }}"

      - alert: GRETunnelHighPacketLoss
        expr: rate(node_network_receive_drop_total{device="gre1"}[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High packet drops on GRE tunnel gre1"
```

## Monitoring Script

```bash
#!/bin/bash
# /usr/local/bin/check-gre-tunnel.sh

TUNNEL="gre1"
REMOTE_IP="172.16.1.2"
THRESHOLD_LOSS=20   # Alert if > 20% packet loss

# Check interface state
STATE=$(ip link show "$TUNNEL" 2>/dev/null | grep -c "LOWER_UP")
if [ "$STATE" -eq 0 ]; then
  echo "CRITICAL: GRE tunnel $TUNNEL interface is DOWN"
  exit 2
fi

# Check ping loss
LOSS=$(ping -c10 -W2 "$REMOTE_IP" 2>&1 | grep -oE "[0-9]+% packet loss" | grep -oE "[0-9]+")
if [ -n "$LOSS" ] && [ "$LOSS" -gt "$THRESHOLD_LOSS" ]; then
  echo "WARNING: GRE tunnel to $REMOTE_IP has ${LOSS}% packet loss"
  exit 1
fi

echo "OK: GRE tunnel $TUNNEL is up, ${LOSS:-0}% loss"
exit 0
```

## Key Takeaways

- `ip -s link show gre1` shows cumulative TX/RX counters; use `watch` to see rate of change.
- GRE interfaces show `state UNKNOWN` normally; check for `LOWER_UP` flag to confirm the tunnel is active.
- Use node_exporter `node_network_up` metric for Prometheus-based GRE tunnel availability monitoring.
- Combine interface state checks with inner-IP ping tests for comprehensive tunnel health monitoring.
