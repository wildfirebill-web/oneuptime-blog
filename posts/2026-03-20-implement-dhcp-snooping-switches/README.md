# How to Implement DHCP Snooping on Switches

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, DHCP Snooping, Network Security, Cisco, Switching, Sysadmin

Description: DHCP snooping is a switch-level security feature that drops DHCP server responses on untrusted ports, preventing rogue DHCP servers from distributing malicious network configuration to clients.

## How DHCP Snooping Works

- **Trusted ports**: Connected to authorized DHCP servers or uplinks. DHCP offers and acks are permitted.
- **Untrusted ports**: Client-facing ports. DHCP offers and acks are dropped; only discovers and requests pass.

The switch builds a DHCP snooping binding table: `{MAC, IP, VLAN, interface}` for each leased address.

## Cisco IOS Configuration

```text
! Enable DHCP snooping globally
ip dhcp snooping

! Enable on specific VLANs
ip dhcp snooping vlan 10,20,30

! Mark uplink to DHCP server as trusted
interface GigabitEthernet0/1
  ip dhcp snooping trust

! Client ports are untrusted by default
! Rate-limit DHCP on client ports to prevent starvation
interface GigabitEthernet0/2
  ip dhcp snooping limit rate 15   ! Max 15 DHCP packets/second

! Verify
show ip dhcp snooping
show ip dhcp snooping binding
```

## Cisco IOS-XE Additional Options

```text
! Insert Option 82 on snooped packets
ip dhcp snooping information option

! Log DHCP snooping violations
ip dhcp snooping database flash:dhcp_snooping.db

! Show snooping statistics
show ip dhcp snooping statistics
```

## Linux: Using iptables to Simulate Snooping

On a Linux bridge/switch:

```bash
# Drop DHCP offers/acks from non-server ports (untrusted)

# eth0 = server port (trusted), eth1/eth2/eth3 = client ports (untrusted)

# Block DHCP server responses from client ports
iptables -A FORWARD -i eth1 -p udp --sport 67 -j DROP
iptables -A FORWARD -i eth2 -p udp --sport 67 -j DROP
iptables -A FORWARD -i eth3 -p udp --sport 67 -j DROP
```

## Verifying Snooping Is Working

```bash
# From a client, try to run a rogue DHCP server
# and confirm it cannot reach clients

# Check Cisco snooping binding table
show ip dhcp snooping binding
# Example output:
# MacAddress       IpAddress    Lease(sec) Type   VLAN Interface
# 00:11:22:33:44:55 10.0.10.50  86400      dynamic 10  Gi0/2
```

## Integration with Dynamic ARP Inspection

DHCP snooping bindings are used by Dynamic ARP Inspection (DAI) to validate ARP packets:

```text
! Enable DAI - validates ARP against DHCP snooping table
ip arp inspection vlan 10,20,30
```

## Key Takeaways

- DHCP snooping marks switch ports as trusted (uplinks, DHCP server) or untrusted (clients).
- Untrusted ports drop any DHCP offer or ack, blocking rogue servers.
- Rate-limit DHCP on client ports to prevent starvation attacks.
- The DHCP snooping binding table feeds Dynamic ARP Inspection for additional security.
