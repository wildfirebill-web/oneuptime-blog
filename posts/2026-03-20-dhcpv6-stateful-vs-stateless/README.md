# How to Understand Stateful vs Stateless DHCPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCPv6, Stateful, Stateless, M Flag, O Flag, IPv6

Description: Understand the difference between stateful DHCPv6 (full address assignment) and stateless DHCPv6 (options only), when to use each, and how RA flags M and O control client behavior.

## Introduction

DHCPv6 operates in two modes: stateful (assigns addresses and tracks state per client) and stateless (provides configuration options without address assignment). The Router Advertisement M and O flags tell clients which mode to use. Stateful DHCPv6 works like DHCPv4 - a server assigns a specific address to each client. Stateless DHCPv6 lets SLAAC handle address generation while DHCPv6 provides DNS and other options.

## Stateful DHCPv6

```text
Stateful DHCPv6 (M=1 in RA):

Server maintains state:
  - Binding table: client DUID → assigned address + lease
  - Tracks which address each client has
  - Renew/rebind/release lifecycle managed

Message exchange:
  SOLICIT → ADVERTISE → REQUEST → REPLY
  (or SOLICIT + RC option → REPLY with RC)

Server stores:
  DUID: 00:01:00:01:12:34:56:78:aa:bb:cc:dd:ee:ff
  IAID: 0x0001  (client's IA identifier)
  Address: 2001:db8::100
  Lease start: 2024-01-15 10:00:00
  T1: 43200s (12 hours)
  T2: 69120s (19.2 hours)
  Valid: 86400s (24 hours)

Client stores:
  Same binding info in lease file

Activation:
  M flag = 1 in RA → client uses stateful DHCPv6 for addresses
```

## Stateless DHCPv6

```text
Stateless DHCPv6 (M=0, O=1 in RA):

Server maintains NO per-client state:
  - No address assignment
  - No lease tracking
  - Same options returned to all clients

Message exchange:
  INFO-REQUEST → REPLY
  (2 messages only)

Server configuration (same options for all):
  DNS: 2001:4860:4860::8888, 2001:4860:4860::8844
  Domain: example.com
  NTP: 2001:db8::ntp1

Client:
  Gets address from SLAAC (RA prefix)
  Gets DNS/options from stateless DHCPv6

Activation:
  M flag = 0 + O flag = 1 in RA
  → client uses SLAAC for address
  → client uses DHCPv6 INFO-REQUEST for options
```

## Comparison Table

```yaml
Stateful vs Stateless DHCPv6:

Feature                  | Stateful (M=1)         | Stateless (M=0, O=1)
-------------------------|------------------------|----------------------
Address assignment       | DHCPv6 server          | SLAAC (from RA)
Server state             | Maintains per-client   | No state
Address tracking         | Full (server log)      | None
DNS assignment           | From DHCPv6 REPLY      | From DHCPv6 REPLY
Custom options           | From DHCPv6 REPLY      | From DHCPv6 REPLY
Address reservations     | Yes (by DUID)          | No
Lease renewal            | Yes (RENEW/REBIND)     | No leases to renew
Server complexity        | High                   | Low
RA required              | Yes (for gateway)      | Yes (for address + gateway)
DHCPv6 messages          | SOLICIT/ADV/REQ/REPLY  | INFO-REQUEST/REPLY only
```

## Configuring M and O Flags

```bash
# radvd: Set M and O flags

# Pure SLAAC (M=0, O=0): no DHCPv6 at all

# interface eth1 {
#     AdvManagedFlag off;    # M=0
#     AdvOtherConfigFlag off; # O=0

# Stateless DHCPv6 (M=0, O=1):
# interface eth1 {
#     AdvManagedFlag off;    # M=0: SLAAC for addresses
#     AdvOtherConfigFlag on;  # O=1: DHCPv6 for options (DNS)

# Stateful DHCPv6 (M=1):
# interface eth1 {
#     AdvManagedFlag on;     # M=1: use stateful DHCPv6 for addresses
#     AdvOtherConfigFlag on;  # O=1: DHCPv6 for other options too
#     # Usually disable SLAAC when M=1:
#     prefix 2001:db8::/64 {
#         AdvAutonomous off; # A=0: don't use this prefix for SLAAC
#     };

# Cisco IOS:
# interface GigabitEthernet0/1
#  ! Stateless DHCPv6:
#  ipv6 nd other-config-flag       ← O=1
#  ! Stateful DHCPv6:
#  ipv6 nd managed-config-flag     ← M=1
#  ipv6 nd other-config-flag       ← O=1 (also set with M=1)
```

## Stateless DHCPv6 Server Configuration (ISC dhcpd)

```bash
# Stateless DHCPv6 server: provides options only, no address assignment
cat > /etc/dhcp/dhcpd6.conf << 'EOF'
# Stateless DHCPv6 configuration
# Provides DNS, domain, NTP but NO addresses

default-lease-time 600;
max-lease-time 7200;

# DHCPv6 subnet configuration
# Note: no "range" statement = no address assignment
subnet6 2001:db8::/64 {
    # Provide options only (stateless)
    option dhcp6.domain-search "corp.example.com" "example.com";
    option dhcp6.name-servers 2001:4860:4860::8888, 2001:4860:4860::8844;
    # No range or host statements (stateless = no address tracking)
}
EOF

# Start ISC dhcpd for DHCPv6
sudo dhcpd -6 -cf /etc/dhcp/dhcpd6.conf eth1

# Stateful DHCPv6 server (add address range):
# subnet6 2001:db8::/64 {
#     range6 2001:db8::100 2001:db8::200;  ← Add this for stateful
#     option dhcp6.domain-search "example.com";
#     option dhcp6.name-servers 2001:4860:4860::8888;
# }
```

## Observing the Difference in Traffic

```bash
# Capture and distinguish stateful vs stateless DHCPv6

# Stateless: only INFO-REQUEST and REPLY
sudo tcpdump -i eth0 -n "udp port 546 or udp port 547" -v | \
    grep "msgtype"
# msgtype: inf-request     ← INFO-REQUEST (stateless)
# msgtype: reply           ← REPLY

# Stateful: SOLICIT, ADVERTISE, REQUEST, REPLY
# msgtype: solicit         ← client looking for server
# msgtype: advertise       ← server offering address
# msgtype: request         ← client requesting address
# msgtype: reply           ← server confirming

# Check host mode: did it use SLAAC or DHCPv6 for address?
ip -6 addr show eth0 | grep "scope global"
# "dynamic" = SLAAC or DHCPv6
# Distinguish: view lease file or check assignment source

# Check DHCPv6 lease file
cat /var/lib/dhclient/dhclient6.leases 2>/dev/null
# If exists and has IA_NA: stateful DHCPv6 used
# If no lease file: SLAAC used (or stateless DHCPv6 for DNS only)
```

## Conclusion

Stateful DHCPv6 (M=1) assigns addresses from a server-managed pool, tracks all bindings, and supports reservations - similar to DHCPv4. Stateless DHCPv6 (M=0, O=1) provides only configuration options (DNS, NTP) while SLAAC handles addresses - simpler and serverless for address tracking. The M and O flags in Router Advertisements control which mode clients use. For most enterprises, stateful DHCPv6 provides the address visibility required for security and compliance. For simpler environments, stateless DHCPv6 or pure SLAAC with RDNSS eliminates server complexity.
