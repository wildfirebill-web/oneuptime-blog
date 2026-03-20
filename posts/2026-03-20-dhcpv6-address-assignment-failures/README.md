# How to Troubleshoot DHCPv6 Address Assignment Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCPv6, IPv6, Troubleshooting, Networking, Diagnostics

Description: A systematic guide to diagnosing and resolving DHCPv6 address assignment failures using logs, packet captures, and configuration checks.

## Overview

When a DHCPv6 client fails to receive an IPv6 address, there are several layers to check: network connectivity, server configuration, firewall rules, and DUID issues. This guide provides a structured troubleshooting approach.

## Troubleshooting Checklist

```mermaid
graph TD
    A[Client not getting IPv6 address] --> B{Is the DHCPv6 server running?}
    B -->|No| C[Start the server: systemctl start kea-dhcp6]
    B -->|Yes| D{Is DHCPv6 traffic reaching the server?}
    D -->|No| E[Check firewall rules on ports 546/547]
    D -->|Yes| F{Is server receiving Solicit?}
    F -->|No| G[Check interface binding in server config]
    F -->|Yes| H{Is server sending Advertise?}
    H -->|No| I[Check pool availability and logs]
    H -->|Yes| J{Is client receiving Advertise?}
    J -->|No| K[Check return path firewall / routing]
    J -->|Yes| L[Check client logs for Request/Reply]
```

## Step 1: Verify the Server is Running

```bash
# Check if Kea DHCPv6 is active
systemctl status kea-dhcp6

# Check ISC dhcpd
systemctl status isc-dhcp-server6

# Verify it's listening on port 547
ss -ulnp | grep 547
```

## Step 2: Check Server Logs

```bash
# Kea logs (adjust path based on your config)
journalctl -u kea-dhcp6 -n 50 --no-pager

# ISC dhcpd logs
tail -f /var/log/syslog | grep dhcpd

# Look for these key log entries:
# "DHCPV6_PACKET_RECEIVED" - server got the Solicit
# "DHCPV6_NO_SUBNET" - no matching subnet found
# "DHCPV6_ALLOCATION_FAIL" - pool exhausted
```

## Step 3: Capture Traffic to Verify Exchange

```bash
# On the server, capture incoming DHCPv6
sudo tcpdump -i eth0 -n -v "udp port 547"

# On the client, trigger a fresh request
sudo dhclient -6 -r eth0 && sudo dhclient -6 eth0 -v

# Expected sequence:
# Client → ff02::1:2: Solicit
# Server → client: Advertise
# Client → server: Request
# Server → client: Reply
```

## Step 4: Check Firewall Rules

```bash
# List current ip6tables rules on the server
sudo ip6tables -L -n -v

# Ensure these rules exist on the server:
# INPUT: -p udp --dport 547 -j ACCEPT
# OUTPUT: -p udp --sport 547 -j ACCEPT

# Quick fix — temporarily flush rules for testing
sudo ip6tables -F  # WARNING: removes all rules
```

## Step 5: Verify Interface Binding

Kea must be configured to listen on the correct interface:

```json
// In kea-dhcp6.conf, verify the interface is listed
{
  "Dhcp6": {
    "interfaces-config": {
      "interfaces": ["eth0"]  // Must match the interface name
    }
  }
}
```

```bash
# Verify the interface name
ip link show
# Restart Kea after fixing
systemctl restart kea-dhcp6
```

## Step 6: Check Subnet Configuration

The client's link-local address must fall within a configured subnet:

```bash
# What link-local address is the client using?
ip -6 addr show dev eth0 | grep "fe80"

# The server must have a subnet matching the interface
# Example: if client is on 2001:db8::/64, server must have that subnet
```

## Step 7: Check Pool Availability

```bash
# Kea: check pool statistics
curl -s -X POST http://localhost:8000/ \
  -H "Content-Type: application/json" \
  -d '{"command": "statistic-get-all", "service": ["dhcp6"]}' | \
  jq '.[0].arguments | to_entries |
      map(select(.key | startswith("subnet"))) |
      map({key, value: .value[0][0]})'
```

## Common Issues and Fixes

| Issue | Symptom | Fix |
|-------|---------|-----|
| Server not listening | No packets on port 547 | Check `interfaces-config` in Kea |
| Pool exhausted | Status code 2 in Reply | Expand pool range or reduce lease time |
| DUID conflict | Client ignores Reply | Clear old lease files on client |
| Wrong subnet | No Advertise | Add matching subnet to server config |
| Firewall blocking | Solicit seen but no Advertise | Add ip6tables rules for port 547 |

## Summary

DHCPv6 failures are almost always caused by one of five issues: the server is not running, firewall rules block ports 546/547, the server is not configured for the right subnet, the pool is exhausted, or there is a DUID mismatch. Work through this checklist in order and use tcpdump to confirm each step.
