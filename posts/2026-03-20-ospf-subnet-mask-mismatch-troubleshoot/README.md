# How to Troubleshoot OSPF Subnet Mask Mismatches

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OSPF, Troubleshooting, IPv4, Subnet Mask, Networking, FRR, Cisco

Description: Learn how to diagnose and fix OSPF neighbor adjacency failures caused by subnet mask mismatches on point-to-point and broadcast links.

---

OSPF requires that two routers on the same link agree on the subnet mask before forming a full adjacency. A subnet mask mismatch is one of the most common causes of OSPF neighbors getting stuck in the `ExStart` or `Down` state.

## How Subnet Mask Mismatches Break OSPF

OSPF Hello packets include the interface's network mask. When two routers have different masks on the same link, they consider themselves on different networks and refuse to form an adjacency.

```text
Router A: 192.168.1.1/24  → Hello contains mask 255.255.255.0
Router B: 192.168.1.2/30  → Hello contains mask 255.255.255.252
                          ↑ Mismatch! Adjacency fails.
```

## Symptoms

- OSPF neighbor stays in `Init` or `Down` state.
- Log messages: `ospf_hello: Packet from 192.168.1.2 network address mismatch`
- `show ip ospf neighbor` shows no neighbor for the expected link.

## Diagnosing the Mismatch

### FRR

```bash
# Show current OSPF neighbors and their state

vtysh -c "show ip ospf neighbor"

# If no neighbor appears for a known link, check interface OSPF settings
vtysh -c "show ip ospf interface eth0"
# Look for: "Network Address 192.168.1.0/24" - compare with the remote router

# Enable OSPF debugging to see Hello packet details
vtysh << 'EOF'
debug ospf hello
debug ospf packet hello detail
EOF

# View debug output in syslog or the FRR log
tail -f /var/log/frr/frr.log | grep -i "network address mismatch"
```

### Cisco IOS

```text
debug ip ospf hello
show ip ospf interface GigabitEthernet0/0
! Look for: Subnet Mask: 255.255.255.0
```

## Common Causes

1. **Typo in prefix length**: One router has `/24`, the other has `/25`.
2. **Secondary addresses**: OSPF is sending hellos from a secondary IP with a different mask.
3. **Loopback assigned as stub**: OSPF treats loopbacks as /32 by default.
4. **VLAN misconfiguration**: The VLAN config on a switch assigns a different network to each router's port.

## Fixing the Mismatch

```bash
# FRR: Correct the interface IP and mask
vtysh << 'EOF'
conf t
interface eth0
  no ip address 192.168.1.2/30
  ip address 192.168.1.2/24
EOF

# Verify the fix
vtysh -c "show interface eth0"
vtysh -c "show ip ospf interface eth0"

# Check neighbor adjacency
vtysh -c "show ip ospf neighbor"
# State should progress to: Init → 2-Way → ExStart → Exchange → Loading → Full
```

## Special Case: OSPF on Unnumbered Interfaces

Point-to-point links can use `ip ospf network point-to-point` to skip the mask check:

```text
interface eth1
  ip ospf network point-to-point
! OSPF will not check subnet masks on this interface
```

## Key Takeaways

- Both routers on an OSPF link must have identical subnet masks for adjacency to form.
- Use `show ip ospf interface` to compare the mask OSPF is advertising vs. what it should be.
- `debug ospf hello` (FRR) or `debug ip ospf hello` (Cisco) reveals mask mismatch errors in real time.
- Use `ip ospf network point-to-point` on transit links to bypass mask checking.
