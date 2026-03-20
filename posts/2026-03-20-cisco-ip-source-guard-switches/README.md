# How to Configure IP Source Guard on Cisco Switches

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cisco, IP Source Guard, IPSG, IPv4, Switching, Security, IOS

Description: Configure IP Source Guard on Cisco IOS switches to prevent IP address spoofing by restricting traffic to only the IP and MAC addresses recorded in the DHCP snooping binding table.

## Introduction

IP Source Guard (IPSG) creates per-port VLAN ACLs that permit only traffic matching the DHCP snooping binding table entries. A host cannot use an IP address it did not obtain from DHCP, blocking IP spoofing at Layer 2.

## Prerequisites

```cisco
! 1. Enable DHCP snooping
ip dhcp snooping
ip dhcp snooping vlan 10

! 2. Populate the binding table (clients must request DHCP leases)
! 3. Trust the uplink
interface GigabitEthernet0/24
 ip dhcp snooping trust
```

## Enable IP Source Guard

```cisco
! Enable IPSG on untrusted access ports
interface GigabitEthernet0/1
 description User-Port
 switchport mode access
 switchport access vlan 10
 ip verify source            ! Filter by IP only

! To also filter by MAC address:
interface GigabitEthernet0/2
 ip verify source port-security
```

## Static Binding for Servers with Static IPs

```cisco
! Servers don't use DHCP - add manual binding
ip source binding 00:1a:2b:3c:4d:5e vlan 10 10.1.10.50 interface GigabitEthernet0/10

! Add multiple static entries
ip source binding 00:aa:bb:cc:dd:ee vlan 10 10.1.10.51 interface GigabitEthernet0/11
```

## Verify IP Source Guard

```cisco
! Show IPSG bindings per interface
show ip verify source

! Example:
! Interface    Filter-type  Filter-mode  IP-address   Mac-address     Vlan
! -----------  -----------  -----------  -----------  --------------  ----
! Gi0/1        ip           active       10.1.10.100  00:1a:2b:3c:4d   10
! Gi0/2        ip-mac       active       10.1.10.101  00:aa:bb:cc:dd   10

! Show DHCP snooping binding table
show ip dhcp snooping binding
```

## IPSG with Port Security

```cisco
interface GigabitEthernet0/3
 switchport mode access
 switchport access vlan 10
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 ip verify source port-security   ! IPSG with MAC filtering
```

## Troubleshooting

```cisco
! Client cannot get IP - check binding table
show ip dhcp snooping binding

! Is DHCP snooping trust set on uplink?
show ip dhcp snooping

! Check IPSG filter mode
show ip verify source interface GigabitEthernet0/1

! Temporarily disable for testing
interface GigabitEthernet0/1
 no ip verify source
```

## Defense in Depth Stack

```text
Layer 2 Security Stack:
  1. Port Security      - limits MAC addresses per port
  2. DHCP Snooping      - validates DHCP, builds binding table
  3. Dynamic ARP Inspection - validates ARP against binding table
  4. IP Source Guard    - validates IP packets against binding table
```

## Conclusion

IP Source Guard is the final layer of the Cisco Layer 2 security stack. It creates per-port dynamic ACLs that drop packets from any IP address not recorded in the DHCP snooping binding table, making IP spoofing impossible on switch access ports. Always add static bindings for hosts using manually configured IPv4 addresses.
