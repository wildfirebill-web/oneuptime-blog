# How to Understand SLAAC Address Lifetimes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SLAAC, Address Lifetime, IPv6, Valid Lifetime, Preferred Lifetime, RFC 4862

Description: Understand IPv6 SLAAC address lifetimes including Valid Lifetime, Preferred Lifetime, DEPRECATED state, and how lifetimes affect address selection and renumbering.

## Introduction

SLAAC addresses have two lifetimes controlled by the Prefix Information option in Router Advertisements: Valid Lifetime and Preferred Lifetime. The Preferred Lifetime determines when an address transitions from PREFERRED to DEPRECATED state. The Valid Lifetime determines when the address is removed entirely. Understanding these lifetimes is essential for IPv6 network renumbering, address stability planning, and troubleshooting connectivity issues.

## Two-Timer Model

```
SLAAC Address Lifetime States:

Timeline:
  T=0       T=PreferredLifetime    T=ValidLifetime
  |_________________|_______________________|
  |    PREFERRED    |      DEPRECATED        |  (after valid: INVALID)

PREFERRED state (0 to PreferredLifetime):
  - Address is fully usable
  - Selected as source address for new connections
  - Normal operation

DEPRECATED state (PreferredLifetime to ValidLifetime):
  - Address is still valid (can receive traffic)
  - Existing connections continue to work
  - NOT selected as source for new connections
  - New connections use newer PREFERRED addresses

INVALID state (after ValidLifetime):
  - Address is removed from interface
  - No traffic can use this address
  - Existing connections drop (connection reset)

Typical defaults from RFC 4861:
  PreferredLifetime: 604800 seconds (7 days)
  ValidLifetime:     2592000 seconds (30 days)
  Deprecated window: 30 days - 7 days = 23 days
```

## Viewing Address Lifetimes

```bash
# Linux: show IPv6 addresses with lifetimes
ip -6 addr show eth0

# Example output:
# inet6 2001:db8::211:22ff:fe33:4455/64 scope global dynamic
#    valid_lft 2591894sec preferred_lft 604694sec
#                         ^^^^^^^^^^^^^^^^^^^^^^^^
#                         preferred_lft = 604694 seconds ≈ 7 days remaining
# valid_lft = 2591894 seconds ≈ 30 days remaining

# Show deprecated addresses (preferred_lft = 0 but valid_lft > 0)
ip -6 addr show | grep "preferred_lft 0"
# inet6 2001:db8::old:addr/64 scope global dynamic
#    valid_lft 86400sec preferred_lft 0sec   ← DEPRECATED

# Show lifetime in hours/days format (custom)
ip -6 addr show eth0 | awk '
/valid_lft/ {
    split($2, v, "sec")
    split($4, p, "sec")
    printf "  Valid: %.1f days  Preferred: %.1f days\n",
           v[1]/86400, p[1]/86400
}'
```

## Lifetime Updates from RA

The kernel updates address lifetimes every time an RA is received.

```
RA Lifetime Update Rules (RFC 4862 Section 5.5.3):

Case 1: Received ValidLifetime > 2 hours OR
        Received ValidLifetime >= remaining ValidLifetime:
  → Update both Valid and Preferred lifetimes

Case 2: Received ValidLifetime < 2 hours AND
        Received ValidLifetime < remaining ValidLifetime:
  → Set remaining ValidLifetime to 2 hours (floor protection)
  → This prevents attack: sending RA with ValidLifetime=0

Case 3: ValidLifetime = 0:
  → Prefix is being withdrawn
  → Preferred Lifetime must also be 0
  → Address is deprecated immediately
  → Removed when current valid lifetime expires

Why 2-hour floor?
  Prevents a rogue RA from setting lifetime to 0 and
  immediately invalidating all existing addresses (DoS attack)
```

## Address Lifetime on Different Systems

```bash
# Linux
ip -6 addr show eth0 | grep "valid_lft"

# macOS
# macOS shows "expires" time in absolute seconds since epoch
ifconfig en0 inet6
# Limited lifetime info; use scutil for details

# Windows
Get-NetIPAddress -AddressFamily IPv6 |
    Select IPAddress, ValidLifetime, PreferredLifetime
# ValidLifetime:    Infinite or number of seconds
# PreferredLifetime: Same

# Note: Windows may show "Infinite" for SLAAC addresses
# due to how it handles RA lifetime updates
```

## Configuring Lifetimes on the Router (radvd)

```bash
# Set custom lifetimes in radvd
cat /etc/radvd.conf
# interface eth1 {
#     AdvSendAdvert on;
#
#     prefix 2001:db8::/64 {
#         AdvOnLink on;
#         AdvAutonomous on;
#
#         # Valid Lifetime: 30 days (2592000 seconds)
#         AdvValidLifetime 2592000;
#
#         # Preferred Lifetime: 7 days (604800 seconds)
#         # Must be <= AdvValidLifetime
#         AdvPreferredLifetime 604800;
#     };
# };

# For infinite lifetime:
# AdvValidLifetime infinity;
# AdvPreferredLifetime infinity;

# For short lifetime (easier renumbering):
# AdvValidLifetime 3600;      # 1 hour
# AdvPreferredLifetime 1800;  # 30 minutes
```

## Using Lifetimes for Network Renumbering

```
Renumbering Procedure using SLAAC Lifetimes:

Phase 1: Prepare (announce new prefix)
  - Router begins advertising new prefix 2001:db8:new::/64
  - Keep advertising old prefix 2001:db8:old::/64
  - Hosts now have both addresses (old=PREFERRED, new=PREFERRED)
  - New connections may use either

Phase 2: Deprecate old prefix
  - Set old prefix PreferredLifetime = 0 in RA
  - Old address transitions to DEPRECATED
  - New connections use new address
  - Existing connections using old address still work

Phase 3: Withdraw old prefix
  - Decrease old prefix ValidLifetime to 0
  - Wait for ValidLifetime to expire on all hosts
  - Old address becomes INVALID
  - Router stops advertising old prefix

Phase 4: Complete
  - Only new prefix 2001:db8:new::/64 active
  - Renumbering complete with no connectivity loss

Note: This process can take days (default lifetimes)
For faster renumbering: use shorter ValidLifetime from start
```

## Conclusion

SLAAC address lifetimes control the transition from PREFERRED to DEPRECATED to INVALID states. The Preferred Lifetime determines when an address stops being selected for new connections. The Valid Lifetime determines when the address is completely removed. The kernel's 2-hour floor protection prevents rogue RAs from immediately invalidating addresses. Plan lifetimes according to your renumbering requirements: shorter lifetimes (hours) allow faster renumbering but require more frequent RA updates; longer lifetimes (days/weeks) provide stability but slower renumbering.
