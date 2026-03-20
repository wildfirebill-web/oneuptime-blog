# How to Troubleshoot IPv6 DAD Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DAD, Duplicate Address Detection, Troubleshooting, NDP, Network Diagnostics

Description: Diagnose and fix IPv6 Duplicate Address Detection (DAD) failures that prevent address assignment, including NDP conflicts, rogue devices, and firewall interference.

## Introduction

Duplicate Address Detection (DAD) is the IPv6 process by which a host verifies that no other device on the link is using the same IPv6 address before assigning it. DAD works by sending a Neighbor Solicitation for the tentative address to the solicited-node multicast group. If another host replies with a Neighbor Advertisement, the address is marked "duplicate" and not used. DAD failures can prevent hosts from getting IPv6 addresses.

## Understanding DAD States

```bash
# Show interface addresses including DAD state
ip -6 addr show dev eth0

# Address states:
# tentative  — DAD in progress, not yet usable
# preferred  — DAD passed, address is valid
# deprecated — address is being phased out
# dadfailed  — DAD detected duplicate, address not assigned
# temporary  — privacy extension address

# Show only tentative addresses (DAD in progress)
ip -6 addr show dev eth0 tentative

# Show DAD-failed addresses
ip -6 addr show dev eth0 dadfailed
```

## Detecting a DAD Failure

```bash
# Monitor system log for DAD failures
journalctl -f | grep -i "DAD\|duplicate\|dadfailed"

# Or check dmesg
dmesg | grep -i "ipv6\|DAD\|duplicate"

# Typical DAD failure message:
# eth0: IPv6 duplicate address 2001:db8::1 detected!

# Use tcpdump to watch DAD in progress
sudo tcpdump -i eth0 -v "ip6 proto 58 and ip6[40] == 135 and src ::"
# Packets with src :: are DAD probes
```

## Capturing DAD Traffic

```bash
# Watch full DAD process (solicitation + advertisement if conflict)
sudo tcpdump -i eth0 -v \
    "ip6 proto 58 and (ip6[40] == 135 or ip6[40] == 136)"

# Decode output shows:
# - Source :: (unspecified) = DAD probe (Neighbor Solicitation)
# - Reply with Neighbor Advertisement = CONFLICT DETECTED

# Example DAD probe:
# IP6 :: > ff02::1:ff00:1: ICMP6, neighbor solicitation,
#   who has 2001:db8::1

# Example conflict response:
# IP6 2001:db8::1 > 2001:db8::100: ICMP6, neighbor advertisement,
#   tgt is 2001:db8::1 (conflict!)
```

## Finding the Conflicting Device

```bash
# After detecting a DAD conflict, find the device using the address
# The conflicting device will have the address in its NDP cache

# Check ARP/NDP cache for the conflicting address
ip -6 neigh show | grep "2001:db8::1"

# Use ndisc6 to query the conflicting device
ndisc6 2001:db8::1 eth0

# Scan the network to find the conflict
# The response will include the conflicting device's MAC address
```

## Disabling DAD (Testing Only)

```bash
# Disable DAD completely (NOT recommended for production)
sudo sysctl -w net.ipv6.conf.eth0.dad_transmits=0

# Set number of DAD probes (default is 1)
sudo sysctl -w net.ipv6.conf.eth0.dad_transmits=1

# Optimistic DAD — use address before DAD completes
sudo sysctl -w net.ipv6.conf.eth0.optimistic_dad=1
```

## Firewall Blocking DAD

```bash
# DAD requires sending NDP packets with source ::
# Some firewalls block packets from unspecified source

# Check if ip6tables blocks DAD probes
sudo ip6tables -L OUTPUT -n | grep "icmp6\|ICMPv6"

# Allow DAD probes (source :: Neighbor Solicitation)
sudo ip6tables -A OUTPUT -p icmpv6 \
    --icmpv6-type neighbor-solicitation -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 \
    --icmpv6-type neighbor-solicitation -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 \
    --icmpv6-type neighbor-advertisement -j ACCEPT
```

## Resolving DAD Address Conflicts

```bash
# Option 1: Remove the conflicting static address and use a different one
sudo ip -6 addr del 2001:db8::1/64 dev eth0
sudo ip -6 addr add 2001:db8::100/64 dev eth0

# Option 2: Find and reconfigure the conflicting device
# Use the MAC address from ndisc6 output to identify the device

# Option 3: If SLAAC is generating conflicting addresses, regenerate
# with a different MAC (or use RFC 7217 stable privacy addresses)
sudo sysctl -w net.ipv6.conf.eth0.addr_gen_mode=3

# Addr gen modes:
# 0 = EUI-64 (from MAC address)
# 1 = stable privacy (RFC 7217)
# 2 = temporary (RFC 4941)
# 3 = random stable
```

## Conclusion

DAD failures occur when two devices attempt to use the same IPv6 address on the same link. Diagnose with `ip -6 addr show dadfailed` and `tcpdump` capturing Neighbor Solicitation packets with source `::`. The conflicting device responds to the DAD probe — capture that response to find its MAC address and identify it. Ensure firewalls allow ICMPv6 Neighbor Solicitation/Advertisement packets, as blocking them prevents DAD from completing and leaves addresses stuck in `tentative` state.
