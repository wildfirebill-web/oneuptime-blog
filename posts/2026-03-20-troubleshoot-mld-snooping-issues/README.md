# How to Troubleshoot MLD Snooping Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MLD, Multicast, Troubleshooting, Switching

Description: A systematic guide to diagnosing and resolving MLD snooping problems that cause IPv6 multicast traffic to not reach subscribers or flood all ports.

## Common MLD Snooping Problems

1. **Multicast not reaching subscribers** - Correct groups not in snooping table
2. **Multicast flooding all ports** - Snooping table missing entries
3. **Multicast stops periodically** - Membership timeouts, no querier
4. **NDP/RA breaks after enabling snooping** - Link-local multicast blocked

## Problem 1: Multicast Not Reaching Subscribers

**Symptom**: Multicast traffic from the source never arrives at subscribing hosts, but the hosts are joined.

**Diagnosis**:
```bash
# On Linux bridge: check if the subscriber's port is in the multicast DB

bridge mdb show | grep 'ff3e::stream'
# If empty: snooping table missing the entry

# On Cisco switch
show ipv6 mld snooping groups vlan 100
# If the host isn't listed: check MLD reports

# Check if the switch received the host's MLD report
tcpdump -i eth0 -n 'icmp6 and (ip6[40] == 131 or ip6[40] == 143)'
# Should see periodic reports from the host
```

**Fix**:
```bash
# If no MLD reports: check host is actually joined
ip -6 maddr show dev eth0

# Trigger MLD report manually by leaving and rejoining
ip -6 maddr del ff3e::stream dev eth0
ip -6 maddr add ff3e::stream dev eth0

# If switch not processing reports: verify snooping is enabled
cat /sys/class/net/br0/bridge/multicast_snooping
# Must be 1 (enabled)
```

## Problem 2: Multicast Flooding All Ports

**Symptom**: All ports in the VLAN receive multicast traffic, even those with no subscribers.

**Diagnosis**:
```bash
# Check if snooping is enabled
ip link show br0 type bridge | grep mcast_snooping

# On Cisco
show ipv6 mld snooping
# Look for: MLD Snooping is globally enabled/disabled

# Check if the switch has a designated router (multicast router) port
# Multicast router ports always receive all multicast
bridge mdb show router
```

**Causes and Fixes**:
```bash
# Cause 1: Snooping not enabled
ip link set br0 type bridge mcast_snooping 1

# Cause 2: Unknown multicast group (not in snooping table)
# Switch floods unknown multicast by default
# Enable immediate membership tracking with fast-leave
ip link set br0 type bridge mcast_last_member_interval 100

# Cause 3: All ports are "router ports" (connected to router)
# Only router-facing ports should receive all multicast
bridge mdb del dev br0 router port eth1  # Remove incorrectly flagged router port
```

## Problem 3: Multicast Stops Periodically

**Symptom**: Multicast works for a few minutes then stops, then restarts.

**Diagnosis**:
```bash
# This is typically a querier issue
# When no queries are sent, membership times out
tcpdump -i eth0 -n 'icmp6 and ip6[40] == 130'
# If no queries seen: no MLD querier present

# Check if the switch has a querier configured
cat /sys/class/net/br0/bridge/mcast_querier
# 0 = disabled, 1 = enabled
```

**Fix**:
```bash
# Enable the bridge as MLD querier
ip link set br0 type bridge mcast_querier 1
ip link set br0 type bridge mcast_query_interval 125
ip link set br0 type bridge mcast_membership_interval 260

# On Cisco switch
ipv6 mld snooping vlan 100 querier address 2001:db8::switch

# Verify querier is running
bridge mdb show
cat /sys/class/net/br0/bridge/mcast_querier_intvl
```

## Problem 4: NDP/RA Breaks After Enabling MLD Snooping

**Symptom**: After enabling MLD snooping, devices lose IPv6 connectivity. Router Advertisements stop working.

**Cause**: Some MLD snooping implementations block link-local multicast groups (`ff02::`) if no MLD report is seen for them. NDP uses `ff02::1`, `ff02::2`, and solicited-node groups.

**Fix**:
```bash
# Configure the switch to never apply snooping to link-local groups
# On Cisco
ipv6 mld snooping vlan 100 link-local-groups-suppression

# On Linux bridge: multicast_router_port ensures router ports forward all multicast
bridge mdb add dev br0 router port eth0  # Designate router port

# Alternatively, add static entries for critical groups
bridge mdb add dev br0 port eth0 grp ff02::1
bridge mdb add dev br0 port eth0 grp ff02::2
```

## Checking MLD Snooping Statistics

```bash
# Linux bridge: check multicast stats
cat /sys/class/net/br0/statistics/multicast

# Cisco switch
show ipv6 mld snooping statistics vlan 100

# Check if snooping is actually doing anything (drops)
ip -s link show br0 | grep -A 3 "RX"
```

## MLD Snooping Diagnostic Script

```bash
#!/bin/bash
echo "=== MLD Snooping Diagnostics ==="

echo -e "\n[1] Snooping Enabled?"
cat /sys/class/net/br0/bridge/multicast_snooping

echo -e "\n[2] Querier Enabled?"
cat /sys/class/net/br0/bridge/mcast_querier

echo -e "\n[3] Multicast DB Entries"
bridge mdb show | head -20

echo -e "\n[4] Router Ports"
bridge mdb show router

echo -e "\n[5] Recent MLD Activity (10s)"
timeout 10 tcpdump -i br0 -n 'icmp6 and (ip6[40] == 130 or ip6[40] == 143)' 2>/dev/null
```

## Summary

MLD snooping issues fall into four categories: missing multicast (check snooping table and MLD reports), flooding (check if snooping is enabled and querier exists), periodic outages (enable MLD querier on the switch), and NDP breakage (ensure link-local multicast bypasses snooping). Always verify a querier is present, check the snooping table matches expected groups, and test with `tcpdump` to observe actual MLD messages.
