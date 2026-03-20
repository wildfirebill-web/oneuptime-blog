# How to Monitor IPsec IPv6 Tunnel Status

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPsec, Monitoring, strongSwan, Prometheus

Description: Learn how to monitor IPv6 IPsec tunnel status using command-line tools, SNMP, Prometheus metrics, and automated alerting for tunnel failures.

## Overview

Monitoring IPv6 IPsec tunnels ensures you detect failures, track performance, and maintain audit records. strongSwan provides built-in monitoring via swanctl, SNMP MIBs, and a REST API. For production environments, integrate with Prometheus or Nagios for alerting and dashboards.

## Basic Monitoring Commands

### strongSwan

```bash
# List all active IKE SAs
swanctl --list-sas

# Sample output:
# gw1-to-gw2: #1, ESTABLISHED, IKEv2, gw1.example.com...gw2.example.com
#   local  'gw1.example.com' @ 2001:db8:gw1::1[500]
#   remote 'gw2.example.com' @ 2001:db8:gw2::1[500]
#   AES_CBC-256/HMAC_SHA2_256_128/PRF_HMAC_SHA2_256/ECP_256
#   established 3600s ago, reauth in 25200s
#   site1-site2: #1, reqid 1, INSTALLED, TUNNEL, ESP:AES_GCM_16-256
#     installed 3598s ago, rekeying in 1s, expires in 2s
#     in  SPI c12345ab, 45892 bytes, 721 packets,  42s ago
#     out SPI ab543210, 38422 bytes, 611 packets,  38s ago
#     local  2001:db8:site1::/48
#     remote 2001:db8:site2::/48

# List active connections (configured, not necessarily up)
swanctl --list-conns

# Show statistics
swanctl --stats
```

### Linux XFRM

```bash
# Show all SAs with byte counters
ip -s xfrm state list

# Show SAs for specific tunnel
ip xfrm state list src 2001:db8:gw1::1 dst 2001:db8:gw2::1

# Monitor SA events in real-time
ip xfrm monitor

# Show SPD (security policies)
ip xfrm policy list

# Check if traffic is being encrypted (counters should increase)
watch -n 2 "ip -s xfrm state list | grep -A 3 'spi 0x'"
```

## Shell Script: Tunnel Health Check

```bash
#!/bin/bash
# check-ipv6-vpn.sh — Monitor strongSwan IPv6 tunnels

TUNNEL_NAME="site1-site2"
REMOTE_HOST="2001:db8:site2::1"
ALERT_EMAIL="noc@example.com"

check_tunnel() {
    # Check IKE SA is established
    STATUS=$(swanctl --list-sas 2>/dev/null | grep -c "ESTABLISHED")
    if [ "$STATUS" -eq 0 ]; then
        echo "CRITICAL: No IKEv2 SA established"
        return 1
    fi

    # Check CHILD SA (IPsec SA) is installed
    CHILD=$(swanctl --list-sas 2>/dev/null | grep -c "INSTALLED")
    if [ "$CHILD" -eq 0 ]; then
        echo "CRITICAL: No IPsec CHILD SA installed"
        return 1
    fi

    # Test connectivity through tunnel
    if ! ping6 -c 2 -W 3 "$REMOTE_HOST" > /dev/null 2>&1; then
        echo "CRITICAL: Cannot ping remote host through tunnel"
        return 1
    fi

    echo "OK: Tunnel $TUNNEL_NAME is up and passing traffic"
    return 0
}

if ! check_tunnel; then
    echo "Tunnel DOWN at $(date)" | mail -s "IPv6 VPN Alert" "$ALERT_EMAIL"
    # Attempt restart
    swanctl --initiate child:$TUNNEL_NAME
fi
```

## Prometheus Monitoring

strongSwan exposes metrics via its VICI socket. Use `prometheus-ipsec-exporter`:

```bash
# Install prometheus-ipsec-exporter
pip install prometheus-ipsec-exporter
# Or use the Docker image:
docker run -d -p 9912:9912 -v /var/run/charon.vici:/var/run/charon.vici \
  ghcr.io/dennisstritzke/ipsec-exporter

# Metrics available:
# ipsec_ikesa_established — Number of established IKE SAs
# ipsec_childsa_installed — Number of installed CHILD SAs
# ipsec_bytes_in_total  — Total bytes inbound per SA
# ipsec_bytes_out_total — Total bytes outbound per SA
# ipsec_packets_in_total
# ipsec_packets_out_total
```

### Prometheus Alert Rules

```yaml
# /etc/prometheus/rules/ipsec.yml
groups:
  - name: ipsec_ipv6
    rules:
      - alert: IPsecTunnelDown
        expr: ipsec_ikesa_established == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "IPv6 IPsec tunnel is down"
          description: "No IKE SA established for 2+ minutes"

      - alert: IPsecNoTraffic
        expr: rate(ipsec_bytes_out_total[5m]) == 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "No outbound traffic on IPv6 IPsec tunnel"
```

## SNMP Monitoring

strongSwan supports SNMP via the ipsec-snmp plugin:

```bash
# Install plugin
apt install strongswan-plugin-ipseckey

# Configure in /etc/strongswan.d/charon/ipsec-snmp.conf
ipsec-snmp {
    socket = udp:localhost:161
    community = public
}
```

```bash
# Query via SNMP
# IKEv2 SA table (IPsec MIB: IPSEC-ISAKMP-IKE-DOI-TC)
snmpwalk -v2c -c public localhost .1.3.6.1.4.1.3317.1.2.7

# Tunnel status
snmpget -v2c -c public localhost IPSEC-MIB::ikeSaState.1
```

## Nagios/Icinga Check

```bash
#!/bin/bash
# check_ipsec_ipv6.sh — Nagios plugin

TUNNEL="$1"
SA_COUNT=$(swanctl --list-sas 2>/dev/null | grep -c "ESTABLISHED")

if [ "$SA_COUNT" -gt 0 ]; then
    echo "OK - $SA_COUNT IPv6 IPsec SA(s) ESTABLISHED | sas=$SA_COUNT"
    exit 0
else
    echo "CRITICAL - No IPv6 IPsec SAs established"
    exit 2
fi
```

## Summary

Monitor IPv6 IPsec tunnels with `swanctl --list-sas` (connection state), `ip -s xfrm state list` (byte counters), and `ip xfrm monitor` (real-time events). For production, deploy `prometheus-ipsec-exporter` to expose strongSwan metrics and create Prometheus alerts for `ipsec_ikesa_established == 0`. Shell scripts can perform end-to-end verification by combining SA state checks with `ping6` tests. Always monitor both SA establishment (IKE layer) and traffic flow (IPsec layer) since an established IKE SA doesn't guarantee data is flowing.
