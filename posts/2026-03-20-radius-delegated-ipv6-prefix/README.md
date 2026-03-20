# How to Use the RADIUS Delegated-IPv6-Prefix Attribute

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RADIUS, Delegated-IPv6-Prefix, DHCPv6-PD, IPv6, RFC 4818, AAA

Description: Configure and use the RADIUS Delegated-IPv6-Prefix attribute (RFC 4818) to assign DHCPv6-PD prefixes to authenticated subscribers for home or enterprise routers.

## Delegated-IPv6-Prefix vs Framed-IPv6-Prefix

| Attribute | RFC | Use Case |
|---|---|---|
| Framed-IPv6-Prefix (attr 97) | RFC 3162 | User's own interface address |
| Delegated-IPv6-Prefix (attr 123) | RFC 4818 | Prefix delegated to CPE router via DHCPv6-PD |

A home subscriber typically gets:
- Framed-IPv6-Prefix: `/128` for the WAN interface
- Delegated-IPv6-Prefix: `/56` or `/48` to route to the home network

## Attribute Wire Format (RFC 4818)

```text
Delegated-IPv6-Prefix (Attribute 123):
  Type:    123
  Length:  Variable
  Value:   reserved (1 byte) + prefix-len (1 byte) + prefix (variable)

Example: 2001:db8:user:a000::/56
  Type:   123
  Length: 10 (4 + 1 reserved + 1 prefix-len + 4 meaningful prefix bytes for /56)
  Byte 3: 0x00 (reserved)
  Byte 4: 0x38 (56 decimal = /56)
  Bytes 5+: 20 01 0d b8 00 00 a0 00 (first 7 bytes, /56 = 7 bytes)
```

## FreeRADIUS Configuration

```text
# /etc/freeradius/3.0/users

# Assign delegated prefix to subscriber (CPE router)

subscriber1  Cleartext-Password := "secret"
             Service-Type = Framed-User,
             Framed-IPv6-Prefix = "2001:db8:wan::1/128",
             Delegated-IPv6-Prefix = "2001:db8:home:a0::/56",
             Framed-IPv6-Route = "2001:db8:home:a0::/56 ::"
```

```sql
-- SQL: store delegated prefix per subscriber
INSERT INTO radreply (username, attribute, op, value) VALUES
('subscriber1', 'Framed-IPv6-Prefix',    '=', '2001:db8:wan::1/128'),
('subscriber1', 'Delegated-IPv6-Prefix', '=', '2001:db8:home:a0::/56'),
('subscriber1', 'Framed-IPv6-Route',     '=', '2001:db8:home:a0::/56 ::');
```

## Cisco BNG: DHCPv6-PD with RADIUS

```text
! Cisco IOS XE BNG - DHCPv6-PD from RADIUS
! RADIUS returns Delegated-IPv6-Prefix, BNG performs DHCPv6-PD toward CPE

ipv6 dhcp pool DELEGATED_POOL
 prefix-delegation aaa
 ! "aaa" = use RADIUS-returned Delegated-IPv6-Prefix

interface Virtual-Template1
 ipv6 enable
 ipv6 dhcp server DELEGATED_POOL rapid-commit

! Subscriber gets:
! WAN address from Framed-IPv6-Prefix
! Delegated /56 from Delegated-IPv6-Prefix (via DHCPv6-PD to CPE)

show ipv6 dhcp binding
! Client: FE80::CPE_LINK_LOCAL
!   IA PD: IA_ID 0x00000001
!     Prefix: 2001:db8:home:a0::/56 valid 86400 preferred 43200
```

## Juniper MX BNG: Delegated Prefix

```text
# Juniper MX - DHCPv6-PD with RADIUS-assigned prefix

set access address-assignment pool DELEGATED_POOL
    family inet6
    prefix-length 56    # /56 per subscriber
    range SUBSCRIBERS prefix 2001:db8:home::/40
    dhcp-attributes
        prefix-length 56
        valid-lifetime 86400
        preferred-lifetime 43200

# RADIUS override: use Delegated-IPv6-Prefix from RADIUS instead of pool
set access profile RADIUS_PROF radius
    accounting accounting-server 2001:db8::radius
    options
        coa-dynamic-request-enable

# Verify DHCPv6-PD bindings
show dhcpv6 server binding detail
```

## Linux: ISC Kea + FreeRADIUS Integration

```bash
# Kea DHCPv6 server uses RADIUS for prefix delegation
# /etc/kea/kea-dhcp6.conf (excerpt)

{
    "Dhcp6": {
        "hooks-libraries": [
            {
                "library": "/usr/lib/kea/hooks/libdhcp_radius.so",
                "parameters": {
                    "server": "2001:db8::radius",
                    "port": 1812,
                    "secret": "radiussecret",
                    "nas-identifier": "bng-1",
                    "extract-duid": true
                }
            }
        ],
        "subnet6": [
            {
                "subnet": "2001:db8::/32",
                "pd-pools": [
                    {
                        "prefix": "2001:db8:home::/40",
                        "prefix-len": 40,
                        "delegated-len": 56
                    }
                ]
            }
        ]
    }
}
```

## FreeRADIUS Dynamic Prefix Pool

```bash
# /etc/freeradius/3.0/mods-enabled/delegated_pool
# Allocate unique /56 per subscriber from a /40 pool

ippool delegated_ipv6 {
    # Store allocations in Redis
    backend = redis
    redis {
        server = "[::1]:6379"
        prefix = "delegated_"
    }

    range-start = 2001:db8:home:0000::/56
    range-stop  = 2001:db8:home:ff00::/56
    prefix-len  = 56

    # Pool size: 2^8 = 256 possible /56 prefixes from a /48
    lease-duration = 2592000  # 30 days
}
```

## Accounting: Tracking Delegated Prefixes

```bash
# FreeRADIUS logs Delegated-IPv6-Prefix in accounting
# /etc/freeradius/3.0/mods-config/sql/main/mysql/queries.conf

# radacct table should store delegated prefix
# Add column if missing:
mysql -u radius -p radius << 'EOF'
ALTER TABLE radacct
    ADD COLUMN delegatedipv6prefix VARCHAR(45) DEFAULT NULL;
EOF

# Query active delegations
mysql -u radius -p radius << 'EOF'
SELECT username, framedipv6prefix, delegatedipv6prefix, acctstarttime
FROM radacct
WHERE acctstoptime IS NULL
  AND delegatedipv6prefix IS NOT NULL
ORDER BY acctstarttime;
EOF
```

## Testing Delegated Prefix Assignment

```bash
# Test with radclient
radclient -x [2001:db8::radius]:1812 auth testing123 << 'EOF'
User-Name = "subscriber1"
User-Password = "secret"
NAS-IPv6-Address = "2001:db8:bng::1"
NAS-Port = 100
Service-Type = Framed-User
EOF

# Expected Access-Accept:
#   Framed-IPv6-Prefix = 2001:db8:wan::1/128
#   Delegated-IPv6-Prefix = 2001:db8:home:a0::/56
#   Framed-IPv6-Route = 2001:db8:home:a0::/56 ::

# Simulate accounting start with delegated prefix
radclient -x [2001:db8::radius]:1813 acct testing123 << 'EOF'
User-Name = "subscriber1"
Acct-Status-Type = Start
Acct-Session-Id = "session-001"
NAS-IPv6-Address = "2001:db8:bng::1"
Framed-IPv6-Prefix = "2001:db8:wan::1/128"
Delegated-IPv6-Prefix = "2001:db8:home:a0::/56"
EOF
```

## Conclusion

Delegated-IPv6-Prefix (RADIUS attribute 123, RFC 4818) enables RADIUS-based control of DHCPv6-PD prefix delegation. Each subscriber receives a unique prefix (typically /56 for residential, /48 for enterprise) that the BNG delegates to the CPE router via DHCPv6-PD. Configure static assignments in the FreeRADIUS SQL `radreply` table or use the `ippool` module for automatic allocation from a pool. Cisco BNG applies the delegated prefix to the DHCPv6-PD pool (`prefix-delegation aaa`), while Juniper MX uses RADIUS CoA for dynamic updates. Always include `Framed-IPv6-Route` alongside the delegated prefix to ensure the BNG installs a route for the delegated block.
