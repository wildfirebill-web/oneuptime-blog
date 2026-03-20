# How to Understand SLAAC Address Deprecation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SLAAC, Address Deprecation, IPv6, Address Lifecycle, Preferred Lifetime

Description: Understand how IPv6 SLAAC addresses transition through PREFERRED and DEPRECATED states, how deprecation affects source address selection, and how to trigger and verify deprecation.

## Introduction

Address deprecation is an important concept in IPv6 SLAAC address management. A DEPRECATED address has exceeded its Preferred Lifetime but remains within its Valid Lifetime. Deprecated addresses continue to receive and transmit traffic for existing connections, but the source address selection algorithm avoids using them for new connections. This mechanism enables graceful network renumbering and address rotation without breaking existing sessions.

## DEPRECATED State Definition

```text
Address States and Source Address Selection:

PREFERRED (active):
  - Preferred Lifetime has NOT expired
  - Source address selection: PREFERRED addresses win Rule 3
    (RFC 6724 source address selection: prefer PREFERRED over DEPRECATED)
  - Default choice for new outbound connections

DEPRECATED:
  - Preferred Lifetime HAS expired (but Valid Lifetime has not)
  - Source address selection: DEPRECATED addresses lose to PREFERRED
  - Existing connections: continue working (address still valid)
  - New connections: use PREFERRED addresses if available
  - Can still receive inbound connections on this address

INVALID:
  - Valid Lifetime HAS expired
  - Address is removed from the interface
  - No traffic possible on this address

Source Address Selection Rule 3 (RFC 6724):
  "Prefer appropriate scope" - but for same scope:
  "Avoid deprecated addresses when a non-deprecated address exists"
  Deprecated = lower preference for new connection source
```

## Viewing Deprecated Addresses

```bash
# Linux: deprecated addresses show preferred_lft 0sec

ip -6 addr show eth0
# inet6 2001:db8::211:22ff:fe33:4455/64 scope global dynamic deprecated
#    valid_lft 86400sec preferred_lft 0sec
#                                    ^^^^ preferred = 0 = DEPRECATED

# Filter: show only deprecated addresses
ip -6 addr show | awk '/preferred_lft 0sec/{found=1} /inet6/{if(found){print; found=0}}'

# Show all addresses with state labels
ip -6 addr show | grep -E "inet6|deprecated|tentative"

# Linux "deprecated" keyword in output confirms DEPRECATED state

# macOS
ifconfig en0 inet6 | grep deprecated
# inet6 2001:db8::old:addr prefixlen 64 autoconf deprecated
#                                                  ^^^^^^^^^^

# Windows
Get-NetIPAddress -AddressFamily IPv6 |
    Where-Object { $_.AddressState -eq "Deprecated" }
```

## Triggering Deprecation (Router Side)

The router triggers deprecation by sending an RA with Preferred Lifetime = 0 for a prefix.

```bash
# radvd: deprecate a prefix by setting PreferredLifetime = 0
# Edit /etc/radvd.conf:
# prefix 2001:db8::/64 {
#     AdvOnLink on;
#     AdvAutonomous on;
#     AdvValidLifetime 86400;      # Keep valid for 1 day
#     AdvPreferredLifetime 0;      # DEPRECATED immediately
# };

# Cisco IOS: deprecate a prefix
# interface GigabitEthernet0/1
#   ipv6 nd prefix 2001:db8::/64 86400 0
#                                        ^
#                                preferred = 0 = deprecated

# The next RA sent after this change will:
# - Include the prefix with preferred lifetime = 0
# - Hosts will immediately mark their address from this prefix as DEPRECATED
# - Existing connections will continue (valid lifetime = 86400sec)
# - New connections will use other PREFERRED addresses
```

## Manually Deprecating an Address on Linux

For testing or manual management, you can manually deprecate an address.

```bash
# Change an address to DEPRECATED state manually
# (set preferred_lft to 0)
sudo ip -6 addr change 2001:db8::211:22ff:fe33:4455/64 \
    dev eth0 preferred_lft 0

# Verify it's deprecated
ip -6 addr show eth0
# inet6 2001:db8::211:22ff:fe33:4455/64 scope global dynamic deprecated
#    valid_lft 86400sec preferred_lft 0sec

# Test: deprecated address is not used for new connections
# (a new PREFERRED address should be present on the interface first)
curl -v --interface eth0 https://example.com
# Source should be the PREFERRED address, not the deprecated one
```

## Deprecation and Source Address Selection Testing

```bash
# Scenario: Two addresses on eth0
#   2001:db8::1 - PREFERRED (valid_lft=86400, preferred_lft=600)
#   2001:db8::2 - DEPRECATED (valid_lft=3600, preferred_lft=0)

# Verify source address selection uses PREFERRED
strace -e trace=connect curl -6 https://ipv6.example.com 2>&1 | \
    grep "sin6_addr"
# Should show 2001:db8::1 as source (PREFERRED)

# Manual test with ip6tables LOG
sudo ip6tables -A OUTPUT -s 2001:db8::2 \
    -j LOG --log-prefix "USING-DEPRECATED: "
# If any packets are logged: deprecated address was used (unexpected)

# More direct test: check which source address is used
ss -6 -n | grep ":443"
# After curl: shows source IP of the connection
```

## Deprecation in Address Rotation (Privacy Extensions)

```text
Privacy Extensions Deprecation Cycle:

Day 0: New temporary address 2001:db8::AAAA generated (PREFERRED)
       Old temporary address 2001:db8::9999 (deprecated)

Day 1 (TEMP_PREFERRED_LIFETIME expires):
  2001:db8::AAAA transitions to DEPRECATED
  New temporary address 2001:db8::BBBB generated (PREFERRED)

Day 7 (TEMP_VALID_LIFETIME expires):
  2001:db8::AAAA transitions to INVALID → removed from interface
  2001:db8::BBBB still PREFERRED

Effect on connections:
  Existing connection using 2001:db8::AAAA:
  - Works fine while still in DEPRECATED state
  - Will break when address becomes INVALID (day 7)
  - Browser will reconnect using 2001:db8::BBBB
```

## Practical Impact: When to Expect Issues

```text
Deprecation issues in practice:

Issue 1: Long-lived connections break after ValidLifetime
  Example: Persistent SSH session, long download
  When DEPRECATED → INVALID: TCP connection resets
  Mitigation: Use longer ValidLifetime for stable addresses
  Or: Use RFC 7217 stable addresses (don't expire if RA keeps them alive)

Issue 2: Application binding to specific deprecated address
  Some applications bind to a specific IPv6 address
  If that address is deprecated then invalidated: app loses connectivity
  Mitigation: Bind to :: (all addresses) not a specific address

Issue 3: Firewall rules matching deprecated address
  Firewall rules using explicit IPv6 addresses may miss the transition
  Mitigation: Use dynamic address objects in firewall rules
```

## Conclusion

Address deprecation is a graceful IPv6 mechanism for transitioning addresses without breaking existing connections. When a SLAAC address's Preferred Lifetime expires, it becomes DEPRECATED - still valid for existing connections but skipped for new ones. The source address selection algorithm (RFC 6724) ensures new connections use PREFERRED addresses when available. Routers trigger deprecation by sending RA with Preferred Lifetime = 0. This mechanism enables clean network renumbering by deprecating old prefixes before withdrawing them.
