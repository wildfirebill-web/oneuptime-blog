# How to Configure IPv6 ND Inspection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ND Inspection, NDP Security, IPv6 Security, First Hop Security, Neighbor Discovery

Description: Configure IPv6 Neighbor Discovery (ND) Inspection to validate NDP messages on managed switches, building a binding table and preventing NA spoofing attacks.

## Introduction

IPv6 ND Inspection (also called IPv6 Snooping on Cisco or ND Security on Juniper) is a switch feature that inspects all NDP messages in hardware as they transit the switch. It validates message content, builds a binding table of address-to-port-to-MAC mappings, and can drop spoofed NDP messages. ND Inspection provides the binding table that IPv6 Source Guard uses for data-plane enforcement, and complements RA Guard (which operates on the control plane).

## What ND Inspection Examines

```text
NDP Messages Inspected by ND Inspection:

Router Solicitation (Type 133):
  - Source address (must not be unspecified in most cases)
  - Source link-layer address option (SLLA)
  - Port role (trusted/untrusted)

Router Advertisement (Type 134):
  - Handled by RA Guard (separate feature)
  - ND Inspection also validates hop limit = 255

Neighbor Solicitation (Type 135):
  - Source address and SLLA option
  - Target address (cannot be multicast)
  - If src=unspecified: DAD in progress (allowed, no binding update)
  - If src=unicast: NS for address resolution (update binding)

Neighbor Advertisement (Type 136):
  - Source address and TLLA option
  - Solicited flag, Override flag
  - Update binding table when valid NA seen

Redirect (Type 137):
  - Source must be valid router from binding table
  - Redirect target must be on-link
```

## Cisco IOS IPv6 Snooping (ND Inspection)

On Cisco, ND Inspection is enabled through the IPv6 Snooping policy.

```text
! Create IPv6 Snooping policy (ND Inspection)
ipv6 snooping policy ND_INSPECT
 security-level guard       ← Validate and drop invalid NDP
 tracking enable            ← Track address-to-port bindings
 limit address-count 10     ← Max 10 IPv6 addresses per port

! Alternative security levels:
!   inspect: validate but do not drop (log only)
!   guard: validate and drop invalid messages

! Apply snooping to VLAN (all ports in VLAN are inspected)
vlan configuration 10
 ipv6 snooping attach-policy ND_INSPECT

! Or apply to specific interface
interface GigabitEthernet1/0/1
 ipv6 snooping attach-policy ND_INSPECT

! Mark trusted ports (not inspected, or inspected differently)
interface GigabitEthernet1/0/24
 ipv6 snooping trust          ← Uplink/router port

! View binding table built by ND Inspection
show ipv6 neighbor binding
```

## Binding Table Lifecycle

```text
Binding Table State Machine:

State: INCOMPLETE
  - NS sent for target, waiting for NA
  - Address not yet confirmed reachable

State: REACHABLE
  - Valid NA received, address confirmed
  - MAC address and port recorded
  - Timer running (default: 300 seconds)

State: STALE
  - Reachable timeout expired
  - Entry kept but marked stale
  - Will be refreshed on next NDP exchange

State: DOWN
  - Interface went down
  - Entry preserved for some time

Binding Table Entry Contains:
  Interface: Gi1/0/1
  VLAN: 10
  IPv6 Address: 2001:db8::1
  MAC Address: 0011.2233.4455
  State: REACHABLE
  Age: 245 sec
  Origin: ND (learned via NDP)
```

## NA Spoofing Prevention

ND Inspection can detect and drop spoofed Neighbor Advertisements.

```text
NA Spoofing Attack:

Attacker sends unsolicited NA:
  src: fe80::attacker
  target: 2001:db8::victim     ← Claims to own victim's address
  TLLA: attacker's MAC          ← Poisons neighbor cache
  Override=1, Solicited=0

ND Inspection defense:
  1. Check binding table: is 2001:db8::victim bound to this port?
  2. If binding shows 2001:db8::victim is on port Gi1/0/5,
     but NA arrives on Gi1/0/1 → SPOOF DETECTED → DROP
  3. Log the violation

Configuration to enable NA validation:
  ipv6 snooping policy ND_INSPECT
   security-level guard   ← Required for drop on violation
```

## ND Inspection Statistics and Verification

```text
! Show binding table
show ipv6 neighbor binding

! Show ND Inspection statistics
show ipv6 snooping counters

! Show snooping policy details
show ipv6 snooping policy ND_INSPECT

! Show snooping on interface
show ipv6 snooping interface GigabitEthernet1/0/1

! Show binding table for specific VLAN
show ipv6 neighbor binding vlan 10

! Example statistics output:
! IPv6 Snooping Counters for GigabitEthernet1/0/1:
!   Messages received: 256
!   Messages validated: 250
!   Messages dropped: 6
!     Reason: Source not in binding table: 3
!     Reason: NA override attack detected: 2
!     Reason: Invalid hop limit: 1
```

## ND Inspection Address Limits

Limiting the number of IPv6 addresses per port prevents ND exhaustion attacks.

```text
! Limit to 10 IPv6 addresses per port (prevents ND table exhaustion)
ipv6 snooping policy STRICT_INSPECT
 security-level guard
 limit address-count 10    ← Adjust based on expected addresses per host
                           ← Normal host: 2-4 addresses (link-local + globals)
                           ← VMs/containers: may need higher limit

! Apply stricter limit to access ports
interface range GigabitEthernet1/0/1 - 23
 ipv6 snooping attach-policy STRICT_INSPECT

! Without this limit: attacker can send 65536 DAD NS messages,
! filling the binding table and causing legitimate addresses to be evicted
```

## Juniper ND Security (ND Inspection)

On Juniper EX switches, ND Inspection is called ND Security.

```text
# Enable ND security on VLAN (enables inspection of NDP messages)

set vlans v10 forwarding-options nd-security

# Mark trusted port (uplink/router port)
set interfaces ge-0/0/23 unit 0 family ethernet-switching nd-security-trusted

# Set maximum bindings per interface (prevents exhaustion)
set vlans v10 forwarding-options nd-security maximum-bindings 10

# View binding table
show nd-security binding

# View statistics
show nd-security statistics

# View per-interface status
show nd-security interface ge-0/0/1
```

## Conclusion

IPv6 ND Inspection validates NDP messages at the switch level and builds a binding table of address-to-port mappings. It detects and drops spoofed Neighbor Advertisements, DAD attacks, and invalid NDP messages. The binding table it creates is used by IPv6 Source Guard for data-plane validation. On Cisco, ND Inspection is configured via IPv6 Snooping policies; on Juniper, it is the ND Security feature. Together with RA Guard and DHCPv6 Guard, ND Inspection forms the complete IPv6 First Hop Security protection suite.
