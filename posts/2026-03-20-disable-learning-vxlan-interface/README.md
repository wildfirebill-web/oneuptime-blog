# How to Disable Learning on a VXLAN Interface

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: VXLAN, Nolearning, FDB, Linux, EVPN, Overlay Networking, Static FDB

Description: Learn how to disable MAC learning on a Linux VXLAN interface and why you would do so, including the use cases for static FDB entries and EVPN-based MAC distribution.

---

Disabling MAC learning on VXLAN forces all forwarding decisions to use static FDB entries or an EVPN control plane. This provides deterministic, flooding-free VXLAN operation.

## Why Disable Learning

- **Security**: Prevents MAC spoofing attacks from populating false FDB entries.
- **Scale**: Dynamic learning causes flooding storms in large overlays.
- **EVPN integration**: EVPN distributes MAC-to-VTEP mappings via BGP; learning is redundant.
- **Deterministic forwarding**: Useful for testing and environments with known MAC assignments.

## Creating VXLAN with nolearning

```bash
# Disable MAC learning at creation time

ip link add vxlan10 type vxlan \
  id 10 \
  dstport 4789 \
  local 10.0.0.1 \
  nolearning

ip link set vxlan10 up

# Verify nolearning is set
ip -d link show vxlan10
# Output includes: ... nolearning ...
```

## Checking If Learning Is Enabled

```bash
# Check via ip -d link show
ip -d link show vxlan10 | grep -E "learning|nolearning"

# Via /sys
cat /sys/class/net/vxlan10/...
# (No direct sysfs knob for learning; use ip link)
```

## Disabling Learning on an Existing Interface

You cannot toggle `nolearning` on a running interface; recreate it:

```bash
# Save FDB entries
bridge fdb show dev vxlan10 | grep permanent > /tmp/fdb-backup.txt

# Remove and recreate
ip link del vxlan10
ip link add vxlan10 type vxlan id 10 dstport 4789 local 10.0.0.1 nolearning
ip link set vxlan10 up

# Restore static FDB entries
while read entry; do
  bridge fdb add $entry
done < /tmp/fdb-backup.txt
```

## Required: Adding FDB Entries Manually

With `nolearning`, all MAC-to-VTEP mappings must be explicitly configured:

```bash
# BUM traffic flood list (all-zeros entry per remote VTEP)
bridge fdb add 00:00:00:00:00:00 dev vxlan10 dst 10.0.0.2 self permanent
bridge fdb add 00:00:00:00:00:00 dev vxlan10 dst 10.0.0.3 self permanent

# Unicast MAC entries
bridge fdb add aa:bb:cc:dd:ee:01 dev vxlan10 dst 10.0.0.2 self permanent
bridge fdb add aa:bb:cc:dd:ee:02 dev vxlan10 dst 10.0.0.3 self permanent
```

## Integration with EVPN (FRR)

```bash
# FRR BGP with EVPN: automatically populates FDB entries
vtysh << 'EOF'
conf t
router bgp 65001
  address-family l2vpn evpn
    advertise-all-vni
    neighbor 10.0.0.2 activate
  exit-address-family
EOF

# EVPN will inject FDB entries via BGP routes
# No need to manage static entries manually
```

## Verifying No Dynamic Entries Appear

```bash
# After enabling nolearning, send traffic and verify no new entries appear
ping 192.168.100.2

bridge fdb show dev vxlan10 | grep -v permanent
# Should be empty (no learned entries)
```

## Key Takeaways

- `nolearning` prevents the kernel from populating FDB entries from incoming VXLAN packets.
- Without learning, you must provide FDB entries statically or via EVPN - otherwise unknown traffic is dropped.
- `nolearning` is recommended for production deployments using EVPN, where BGP distributes all MAC-IP bindings.
- Disabling learning also prevents MAC flooding attacks from propagating false VTEP associations.
