# How to Inspect the VXLAN FDB (Forwarding Database)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, VXLAN, FDB, Forwarding Database, Networking, Troubleshooting, Overlay Network

Description: Inspect and manage the VXLAN forwarding database to see MAC-to-VTEP mappings, verify MAC learning, and add static entries for overlay network troubleshooting.

## Introduction

The VXLAN FDB (Forwarding Database) maps MAC addresses to remote VTEP IP addresses. The bridge/kernel uses this table to decide where to send VXLAN-encapsulated frames. Dynamic entries are learned from traffic; static entries are added manually. Inspecting the FDB is essential for troubleshooting VXLAN connectivity.

## View the VXLAN FDB

```bash
# Show all FDB entries for the VXLAN interface

bridge fdb show dev vxlan0

# Example output:
# 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.2 self permanent
# 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.3 self permanent
# aa:bb:cc:dd:ee:11 dev vxlan0 dst 10.0.0.2 self
```

## Interpret FDB Entries

```text
00:00:00:00:00:00 dev vxlan0 dst 10.0.0.2 self permanent
```
- `00:00:00:00:00:00` - zero MAC = flood/BUM entry
- `dst 10.0.0.2` - send BUM traffic to this VTEP
- `permanent` - static entry (won't age out)

```text
aa:bb:cc:dd:ee:11 dev vxlan0 dst 10.0.0.2 self
```
- Real MAC address = dynamically learned unicast entry
- `dst 10.0.0.2` - this MAC is reachable via this VTEP
- No `permanent` = dynamic (will age out)

## Add a Static MAC-to-VTEP Entry

```bash
# Add a permanent unicast entry for a known remote MAC
bridge fdb add aa:bb:cc:dd:ee:11 dev vxlan0 dst 10.0.0.2 permanent

# Add a flood entry for a new remote VTEP
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.4 permanent
```

## Delete an FDB Entry

```bash
# Delete a specific FDB entry
bridge fdb del aa:bb:cc:dd:ee:11 dev vxlan0 dst 10.0.0.2

# Delete a flood entry
bridge fdb del 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.2
```

## Monitor FDB Learning in Real Time

```bash
# Generate VXLAN traffic on another terminal
ping -c 10 10.200.0.2 &

# Watch FDB entries appear as MACs are learned
watch -n 1 "bridge fdb show dev vxlan0 | grep -v permanent"
```

## FDB Aging

Dynamic FDB entries age out. Check and tune aging time:

```bash
# Check aging time of the VXLAN interface (via bridge it's attached to)
cat /sys/class/net/br-overlay/bridge/ageing_time

# Set aging time (seconds × 100 in kernel units, or set via networkd)
ip link set br-overlay type bridge ageing_time 300
```

## FDB and ARP Suppression

For large overlays, check if ARP suppression is enabled (reduces flooding):

```bash
# Check ARP suppression
ip -d link show vxlan0 | grep "arp_suppress"

# Enable ARP suppression
ip link set vxlan0 type vxlan arp_proxy 1
```

## Conclusion

The VXLAN FDB is the MAC-to-VTEP mapping table that drives forwarding decisions. Use `bridge fdb show dev vxlan0` to inspect it. Zero-MAC entries are flood/BUM entries; real MAC entries are unicast forwarding entries. Dynamic entries are learned from traffic and will age out; permanent entries remain until explicitly deleted. Inspecting the FDB is the primary tool for diagnosing VXLAN connectivity issues.
