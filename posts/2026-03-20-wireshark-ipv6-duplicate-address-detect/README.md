# How to Analyze IPv6 Duplicate Address Detection in Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wireshark, IPv6, Duplicate Address Detection, DAD, NDP, ICMPv6

Description: A guide to capturing and analyzing IPv6 Duplicate Address Detection (DAD) exchanges in Wireshark to diagnose address conflicts and failed DAD processes.

Duplicate Address Detection (DAD) is an IPv6 mechanism that verifies a candidate address is not already in use before assigning it to an interface. DAD uses special ICMPv6 Neighbor Solicitation messages. Wireshark can decode all DAD-related packets.

## How DAD Works

1. Before assigning an IPv6 address, the host sends a **Neighbor Solicitation** with:
   - Source: `::` (unspecified address)
   - Destination: the address's solicited-node multicast group
   - Target: the candidate IPv6 address

2. If no response is received within 1 second, the address is unique and gets assigned.

3. If a **Neighbor Advertisement** is received, there is a duplicate — the address is NOT assigned.

## Display Filters for DAD

```wireshark
# Show ALL DAD-related Neighbor Solicitations
# DAD NS packets have source address ::
icmpv6.type == 135 && ipv6.src == ::

# Show all Neighbor Advertisements that could be DAD conflicts
# NA sent to all-nodes (not unicast) is suspicious
icmpv6.type == 136 && ipv6.dst == ff02::1

# Show both DAD NS and potential conflict NA together
(icmpv6.type == 135 && ipv6.src == ::) ||
(icmpv6.type == 136 && ipv6.dst == ff02::1)
```

## Identify DAD Conflict

A DAD conflict occurs when a DAD NS is followed by an NA for the same target:

```wireshark
# Show DAD messages for a specific candidate address
(icmpv6.type == 135 && ipv6.src == :: &&
 icmpv6.nd.ns.target_address == 2001:db8::10)
||
(icmpv6.type == 136 &&
 icmpv6.nd.na.target_address == 2001:db8::10)
```

In the packet timeline:
1. Look for a **NS** with source `::` targeting `2001:db8::10`
2. If followed by a **NA** targeting `2001:db8::10`, there is a duplicate

## Normal DAD Sequence (No Conflict)

A successful DAD for address `2001:db8::10`:

```
Time  | Src         | Dst                  | Info
0.000 | ::          | ff02::1:ff00:0010    | NS target=2001:db8::10
1.000 | [silence]                          | No reply -> address is unique!
1.000 | 2001:db8::10| ff02::1              | NA (now announcing the address)
```

## DAD Failure Sequence (Conflict)

```
Time  | Src              | Dst               | Info
0.000 | ::               | ff02::1:ff00:0010 | NS target=2001:db8::10 (DAD probe)
0.005 | 2001:db8::10     | ff02::1           | NA (conflict! address already in use)
      | [host A does NOT assign the address]
```

## Capture DAD Traffic

```bash
# Capture only ICMPv6 traffic (includes all NDP and DAD)
sudo tcpdump -i eth0 icmp6 -w dad-capture.pcap

# Filter to show only DAD probes
sudo tcpdump -i eth0 'ip6 src :: and icmp6[0] == 135'
```

## Diagnosing DAD Issues

### No Address Being Assigned

Check if DAD probes are being sent and if any conflict responses are received:

```bash
# Watch for DAD probes on the interface
sudo ip monitor neigh | grep "INCOMPLETE\|FAILED"

# Check kernel message for DAD failures
dmesg | grep -i "duplicate"
```

### Spurious DAD Conflicts

If a host frequently gets DAD conflicts but the conflicting host doesn't exist, you may have:
- A rogue device responding to NDP
- A misconfigured virtual machine with a duplicate MAC-derived EUI-64 address
- IPv6 privacy extension addresses colliding

```wireshark
# Find what device is causing the DAD conflict
# Look at the NA that triggers the failure
icmpv6.type == 136 && icmpv6.nd.na.target_address == <conflicting-address>

# The Ethernet source MAC in the NA is the conflicting device
eth.src
```

## Verify DAD is Working Correctly

```bash
# Check current IPv6 address states (DADFAILED = bad)
ip -6 addr show | grep -E "TENTATIVE|DADFAILED|permanent"

# Manually trigger DAD for all addresses
ip -6 addr flush dev eth0 scope global
dhclient -6 eth0   # Or let SLAAC reconfigure
```

Wireshark's ability to filter for the unspecified source address `::` makes it uniquely effective for isolating DAD traffic and pinpointing the exact device causing IPv6 address conflicts.
