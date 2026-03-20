# How to Troubleshoot DHCPv6 Relay Problems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCPv6, Relay, Troubleshooting, Diagnostics, Networking

Description: Troubleshoot common DHCPv6 relay problems including clients not getting addresses, relay dropping messages, and server not responding.

## Common DHCPv6 Relay Problems

| Symptom | Most Likely Cause |
|---|---|
| Client stays in INIT-RECONF | No relay or RA M-flag not set |
| Relay forwards but no REPLY | Server unreachable or misconfigured |
| Wrong address from wrong pool | Wrong relay link-address/Option 18 |
| Intermittent failures | Relay dropping under load |
| Address assigned then lost | Short lease / relay restart |

## Problem 1: Clients Not Receiving Addresses

```bash
#!/bin/bash
# diagnose-no-address.sh — Run on relay agent

echo "=== DHCPv6 Relay Diagnosis ==="

# 1. Is relay daemon running?
if systemctl is-active --quiet isc-dhcp-relay6; then
    echo "PASS: Relay daemon running"
else
    echo "FAIL: Relay daemon not running"
    echo "Fix: systemctl start isc-dhcp-relay6"
fi

# 2. Is relay listening on UDP 547?
if ss -6 -ulnp | grep -q ":547"; then
    echo "PASS: Listening on UDP 547"
else
    echo "FAIL: Not listening on UDP 547"
    echo "Fix: Check relay configuration and restart"
fi

# 3. Is multicast group joined?
if ip -6 maddr show eth0 | grep -q ff02::1:2; then
    echo "PASS: Multicast ff02::1:2 joined on eth0"
else
    echo "FAIL: Multicast not joined on eth0"
    echo "Fix: Bring up eth0 and restart relay"
fi

# 4. Is server reachable?
if ping6 -c 3 -W 2 2001:db8::dhcp-server &>/dev/null; then
    echo "PASS: DHCPv6 server reachable"
else
    echo "FAIL: Server 2001:db8::dhcp-server unreachable"
    echo "Fix: Check IPv6 routing to server"
fi

# 5. RA flags on client interface
MANAGED=$(sysctl -n net.ipv6.conf.eth0.forwarding)
echo "INFO: Client interface IPv6 forwarding: ${MANAGED}"
```

## Problem 2: Relay Receiving But Not Forwarding

```bash
# Capture to verify relay is receiving SOLICITs
tcpdump -i eth0 -c 10 -n 'udp port 547'
# Should see: IP6 fe80::client > ff02::1:2: dhcp6 solicit

# Check if relay-forw is being sent out server interface
tcpdump -i eth1 -c 10 -n 'udp port 547'
# Should see: IP6 relay-addr > dhcp-server-addr: dhcp6 relay-forw

# If receiving but not forwarding — check:
# 1. Is the server address correct?
grep -r "dhcp-server" /etc/dhcp/ /etc/wide-dhcpv6/ 2>/dev/null

# 2. Is server interface up?
ip link show eth1

# 3. Is there a route to the server?
ip -6 route get 2001:db8::dhcp-server
```

## Problem 3: Wrong Address Pool Assigned

```bash
# Verify relay link-address matches server subnet configuration

# Check what link-address the relay is using
tcpdump -i eth1 -n -v 'udp port 547' | grep "link-address"

# Expected: relay sends link-address = 2001:db8:1::1 (its client-facing address)
# Server should have a subnet matching this address range

# If wrong: check relay configuration
# dhcrelay: the source address is the relay's interface address
# ISC Kea server: check if subnet matches relay's link-address
cat /etc/kea/kea-dhcp6.conf | python3 -c "
import json, sys
config = json.load(sys.stdin)
for subnet in config['Dhcp6']['subnet6']:
    print(f\"Subnet: {subnet['subnet']}\")
"
```

## Problem 4: Relay Dropping Messages

```bash
# Check relay error counters
# Cisco IOS
# show ipv6 dhcp relay statistics | include [Dd]rop

# Linux — check if relay is hitting resource limits
# Increase relay process limits
cat /proc/$(pgrep dhcrelay)/limits | grep "Max open files"
# If too low: increase ulimit

# Check for duplicate relay configuration
# (Two relay processes on same interface = drops)
pgrep -a dhcrelay

# Check iptables isn't blocking relay
ip6tables -L -n -v | grep DROP | grep 547
```

## Problem 5: Relay Appears Working But Server Has No Bindings

```bash
# This usually means relay is sending to wrong server address
# or server is rejecting relayed messages

# On server: enable debug logging (ISC Kea)
# Set log level to DEBUG in kea-dhcp6.conf
# journalctl -u kea-dhcp6 -f | grep -E "RELAY|subnet|query"

# Common cause: server doesn't have a subnet matching relay link-address
# Server needs: subnet matching the relay's link-address

# Example: relay has link-address 2001:db8:1::1
# Server kea-dhcp6.conf must have:
# {
#   "subnet": "2001:db8:1::/64",
#   "relay": {"ip-addresses": ["2001:db8:1::1"]}
# }

# Check Kea logs for "no subnet found for address"
journalctl -u kea-dhcp6 | grep "no subnet"
```

## Quick Diagnostic Script

```bash
#!/bin/bash
# quick-dhcpv6-relay-check.sh

RELAY_IFACE=${1:-eth0}
SERVER_ADDR=${2:-"2001:db8::dhcp-server"}

PASS=0; FAIL=0

check() {
    local MSG=$1; local CMD=$2
    if eval "${CMD}" &>/dev/null; then
        echo "  PASS: ${MSG}"; ((PASS++))
    else
        echo "  FAIL: ${MSG}"; ((FAIL++))
    fi
}

check "Relay listening UDP 547" "ss -6 -ulnp | grep ':547'"
check "Interface ${RELAY_IFACE} up" "ip link show ${RELAY_IFACE} | grep -q UP"
check "Multicast ff02::1:2 joined" "ip -6 maddr show ${RELAY_IFACE} | grep ff02::1:2"
check "Server ${SERVER_ADDR} reachable" "ping6 -c 2 -W 3 ${SERVER_ADDR}"
check "IPv6 forwarding enabled" "[ \$(sysctl -n net.ipv6.conf.all.forwarding) -eq 1 ]"

echo ""
echo "Result: ${PASS} passed, ${FAIL} failed"
```

## Conclusion

DHCPv6 relay troubleshooting proceeds from client to relay to server. Verify each link in the chain with `tcpdump`: client SOLICIT → relay RELAY-FORW → server RELAY-REPL → client REPLY. The most common issues are: relay daemon not running, server unreachable due to missing IPv6 route, incorrect server subnet configuration (server doesn't have a subnet matching the relay's link-address), and firewall blocking UDP 547. The quick diagnostic script automates the most common checks.
