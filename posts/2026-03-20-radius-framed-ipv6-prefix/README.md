# How to Use the RADIUS Framed-IPv6-Prefix Attribute

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RADIUS, Framed-IPv6-Prefix, IPv6, RFC 3162, Address Assignment, AAA

Description: Configure and use the RADIUS Framed-IPv6-Prefix attribute (RFC 3162, attribute 97) to assign IPv6 addresses and prefixes to authenticated users.

## Framed-IPv6-Prefix Overview

Framed-IPv6-Prefix (RADIUS attribute 97, RFC 3162) assigns an IPv6 prefix to an authenticated user. The NAS (router, switch, BNG) applies this prefix to create the user's IPv6 interface address.

| Use Case | Prefix Length | Example |
|---|---|---|
| Single host address | /128 | 2001:db8:1::a/128 |
| Point-to-point link | /64 | 2001:db8:1:user::/64 |
| Framed route target | /48 or /56 | 2001:db8:user::/48 |

Note: Delegated-IPv6-Prefix (attribute 123) is used for DHCPv6-PD. Framed-IPv6-Prefix is for the user's own interface address.

## Attribute Wire Format

```text
Framed-IPv6-Prefix (Attribute 97):
  Type:    97
  Length:  Variable (4 + prefix bytes)
  Value:   reserved (1 byte) + prefix-len (1 byte) + prefix (N bytes)

Example: 2001:db8:1::10/128
  Type:    97
  Length:  20  (4 header + 16 prefix bytes)
  Byte 3:  0x00 (reserved)
  Byte 4:  0x80 (128 = /128)
  Bytes 5-20: 20 01 0d b8 00 01 00 00 00 00 00 00 00 00 00 10
```

## FreeRADIUS: Assigning Framed-IPv6-Prefix

```text
# /etc/freeradius/3.0/users

# Assign specific /128 to user (like PPP IPv4 Framed-IP-Address)

alice  Cleartext-Password := "secret"
       Framed-IPv6-Prefix = "2001:db8:1::a/128",
       Framed-IPv6-Route = "2001:db8:1::/64 ::",
       Service-Type = Framed-User

# Assign /64 prefix (user gets entire /64)
bob    Cleartext-Password := "secret"
       Framed-IPv6-Prefix = "2001:db8:2:bob::/64",
       Service-Type = Framed-User

# Assign from pool
carol  Cleartext-Password := "secret"
       Framed-IPv6-Pool = "ipv6_users",
       Service-Type = Framed-User
```

## SQL-Based Prefix Assignment

```sql
-- radreply table: static per-user assignments
INSERT INTO radreply (username, attribute, op, value) VALUES
('alice', 'Framed-IPv6-Prefix', '=', '2001:db8:1::a/128'),
('alice', 'Framed-IPv6-Route', '=', '2001:db8:1::/64 ::'),
('bob',   'Framed-IPv6-Prefix', '=', '2001:db8:2:bob::/64');

-- View all assigned prefixes
SELECT username, value AS ipv6_prefix
FROM radreply
WHERE attribute = 'Framed-IPv6-Prefix'
ORDER BY username;
```

## Cisco IOS BNG: Applying Framed-IPv6-Prefix

```text
! Cisco BNG (Broadband Network Gateway)
! RADIUS returns Framed-IPv6-Prefix - applied to subscriber interface

interface Virtual-Template1 type aaa
 ipv6 enable
 peer ipv6 pool RADIUS_ASSIGNED   ! Use RADIUS-assigned address
 no peer default ipv6 address pool

! Policy map to apply RADIUS-returned prefix
policy-map type control subscriber PM_IPv6
 event session-start match-all
  class type control subscriber CM_IPv6 do-until-failure
   1 authorize aaa list RADIUS_LIST

! Verification
show ipv6 subscribers
show subscriber session all detail | include IPv6
```

## Juniper BNG: Applying Framed-IPv6-Prefix

```text
# Juniper MX BNG configuration for Framed-IPv6-Prefix

set access profile RADIUS_PROF authentication-order radius
set access profile RADIUS_PROF radius authentication-server 2001:db8::radius

# Subscriber management
set interfaces ge-0/0/0 unit 0 family inet6 dhcpv6-client
set access address-assignment pool IPV6_POOL family inet6 prefix 2001:db8:users::/48

# Framed-IPv6-Prefix is applied as the subscriber interface address
# Verify:
show subscribers detail
show network-access aaa radius statistics
```

## Linux PPPoE Server with Framed-IPv6-Prefix

```bash
# pppd with RADIUS - applies Framed-IPv6-Prefix to PPP interface

# /etc/ppp/options
plugin radius.so
plugin radattr.so
radius-config-file /etc/radiusclient/radiusclient.conf

# When RADIUS returns Framed-IPv6-Prefix:
# pppd assigns the prefix to the ppp0 interface
# Example: ppp0 gets inet6 2001:db8:1::a/128

# Verify assigned prefix
ip -6 addr show ppp0
# inet6 2001:db8:1::a/128 peer 2001:db8:1::1/128 scope global
```

## IPv6 Pool Configuration in FreeRADIUS

```bash
# /etc/freeradius/3.0/mods-enabled/ippool_ipv6
# Module for IPv6 prefix pool assignment

ippool ipv6_users {
    # Use Redis or SQL backend for pool tracking
    backend = redis

    redis {
        server = "[::1]:6379"
    }

    range-start = 2001:db8:users::1
    range-stop  = 2001:db8:users::ffff
    prefix-len  = 128  # Assign /128 to each user

    lease-duration = 86400  # 24 hours
    return-on-no-address = yes
}

# In authorize section:
authorize {
    ippool ipv6_users
}
```

## Testing Framed-IPv6-Prefix Assignment

```bash
# Test that RADIUS returns Framed-IPv6-Prefix
radclient -x [2001:db8::radius]:1812 auth testing123 << 'EOF'
User-Name = "alice"
User-Password = "secret"
NAS-IPv6-Address = "2001:db8:nas::1"
NAS-Port = 1
EOF

# Expected response:
# Received Access-Accept Id 0 from ...
#   Framed-IPv6-Prefix = 2001:db8:1::a/128
#   Framed-IPv6-Route = 2001:db8:1::/64 ::
#   Service-Type = Framed-User

# Parse the prefix from response
RESPONSE=$(radclient [2001:db8::radius]:1812 auth testing123 <<< "User-Name = alice
User-Password = secret")
PREFIX=$(echo "$RESPONSE" | grep "Framed-IPv6-Prefix" | awk '{print $3}')
echo "Assigned prefix: ${PREFIX}"
```

## Conclusion

Framed-IPv6-Prefix (attribute 97) is the primary mechanism for assigning IPv6 addresses to authenticated users via RADIUS. A /128 behaves like a host address assignment (equivalent to IPv4 Framed-IP-Address), while a /64 gives the user an entire subnet. Configure static assignments in the FreeRADIUS `users` file or SQL `radreply` table, or use the `ippool` module for dynamic assignment from a pool. NAS devices (Cisco BNG, Juniper MX) apply the returned prefix directly to the subscriber interface. Test with `radclient` and verify the attribute appears in the Access-Accept response.
