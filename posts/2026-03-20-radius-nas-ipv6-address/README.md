# How to Use the NAS-IPv6-Address RADIUS Attribute

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RADIUS, NAS-IPv6-Address, IPv6, RFC 3162, AAA, Networking

Description: Configure and use the NAS-IPv6-Address RADIUS attribute (RFC 3162, attribute 95) for identifying network access servers by IPv6 address in authentication and accounting.

## What Is NAS-IPv6-Address?

NAS-IPv6-Address (RADIUS attribute 95, RFC 3162) identifies the IPv6 address of the Network Access Server (NAS) — the device sending RADIUS requests. It replaces NAS-IP-Address (attribute 4) for IPv6 NAS devices.

The NAS includes this attribute when:
- The NAS uses an IPv6 address to connect to the RADIUS server
- The NAS wants to identify itself with its IPv6 management address
- Accounting records need to identify the NAS by IPv6

## NAS-IPv6-Address in Access-Request

```
RADIUS Access-Request packet (from IPv6 NAS):

Attribute 4  (NAS-IP-Address):      Not present (IPv6 NAS)
Attribute 95 (NAS-IPv6-Address):    2001:db8:nas::1
Attribute 5  (NAS-Port):            1
Attribute 1  (User-Name):           alice
Attribute 2  (User-Password):       ****
Attribute 61 (NAS-Port-Type):       Ethernet
```

## Cisco IOS/IOS-XE: Configuring NAS-IPv6-Address

```
! Cisco IOS XE — RADIUS NAS IPv6 identification
ipv6 address 2001:db8:nas::1/128 loopback0

! Configure RADIUS server (IPv6)
radius server RADIUS_SRV
 address ipv6 2001:db8::radius
 key mysecret

! Set NAS identifier (use IPv6 loopback)
ip radius source-interface Loopback0
! This sends NAS-IPv6-Address = 2001:db8:nas::1

! Enable IPv6 AAA
aaa new-model
aaa authentication login default group radius local
aaa authorization network default group radius
```

## Juniper Junos: NAS-IPv6-Address

```
# Junos configuration for IPv6 RADIUS NAS
set access radius-server 2001:db8::radius secret mysecret
set access radius-server 2001:db8::radius port 1812
set access radius-server 2001:db8::radius source-address 2001:db8:nas::1

# The source-address becomes the NAS-IPv6-Address in RADIUS packets
# Verify:
show access radius-server
show network-access aaa statistics radius
```

## Linux/FreeRADIUS: Simulating NAS-IPv6-Address

```bash
# radclient: include NAS-IPv6-Address in test request
cat > /tmp/access-request.txt << 'EOF'
User-Name = "testuser"
User-Password = "testpass"
NAS-IPv6-Address = "2001:db8:nas::1"
NAS-Port = 0
NAS-Port-Type = Ethernet
Calling-Station-Id = "00:11:22:33:44:55"
EOF

# Send to RADIUS server
radclient -x [2001:db8::radius]:1812 auth testing123 < /tmp/access-request.txt

# radclient automatically adds NAS-IPv6-Address from source if not specified
# when connecting via IPv6
radclient -x -S [2001:db8::radius]:1812 auth testing123
# -S = use source address as NAS identifier
```

## FreeRADIUS: Using NAS-IPv6-Address in Policy

```
# /etc/freeradius/3.0/policy.d/nas-ipv6
# Apply different policies based on NAS IPv6 address

policy nas_ipv6_policy {
    if (NAS-IPv6-Address =~ /^2001:db8:site-a:/) {
        # NAS is in Site A
        update reply {
            Framed-IPv6-Prefix := "2001:db8:site-a:users::/64"
        }
    }
    elsif (NAS-IPv6-Address =~ /^2001:db8:site-b:/) {
        # NAS is in Site B
        update reply {
            Framed-IPv6-Prefix := "2001:db8:site-b:users::/64"
        }
    }
    else {
        reject
    }
}
```

## FreeRADIUS: Client Verification Against NAS-IPv6-Address

```
# /etc/freeradius/3.0/clients.conf
# RADIUS client (NAS) defined by IPv6 address

client core_router {
    ipv6addr = 2001:db8:nas::1
    secret   = naspassword
    shortname = core-router
    nastype  = cisco

    # FreeRADIUS verifies packet source IP matches this entry
    # NAS-IPv6-Address in packet should match as well
}

# Virtual server: log which NAS is authenticating
# /etc/freeradius/3.0/sites-enabled/default
server default {
    authorize {
        if (NAS-IPv6-Address) {
            update request {
                Tmp-String-0 := "%{NAS-IPv6-Address}"
            }
            update config {
                Auth-Log-Message := "Auth from NAS %{NAS-IPv6-Address}"
            }
        }
    }
}
```

## SQL Logging of NAS-IPv6-Address

```sql
-- radacct schema includes NAS address columns
-- Verify IPv6 NAS addresses are being stored

SELECT nasipaddress, nasipv6address, username, acctstarttime
FROM radacct
WHERE nasipv6address IS NOT NULL
ORDER BY acctstarttime DESC
LIMIT 10;

-- Count sessions per IPv6 NAS
SELECT nasipv6address, COUNT(*) as sessions
FROM radacct
WHERE acctstoptime IS NULL  -- active sessions
GROUP BY nasipv6address
ORDER BY sessions DESC;
```

```bash
# FreeRADIUS SQL accounting schema — ensure IPv6 columns exist
mysql -u radius -p radius << 'EOF'
ALTER TABLE radacct ADD COLUMN nasipv6address VARCHAR(50) DEFAULT NULL;
ALTER TABLE radpostauth ADD COLUMN nasipv6address VARCHAR(50) DEFAULT NULL;
EOF

# FreeRADIUS mods-config/sql/main/mysql/queries.conf
# accounting_start_query — include NAS-IPv6-Address
```

## Troubleshooting NAS-IPv6-Address

```bash
# Check if NAS-IPv6-Address appears in Access-Request
# Enable FreeRADIUS debug mode
freeradius -X 2>&1 | grep -i "NAS-IPv6"

# Packet capture: view raw RADIUS attribute 95
tcpdump -i eth0 -n udp port 1812 -w /tmp/radius.pcap
tshark -r /tmp/radius.pcap -Y "radius" -T fields \
    -e radius.attr_type \
    -e radius.framed_ipv6_prefix | head -20

# Verify client block matches NAS IPv6 address
radtest -x testuser testpass 2001:db8::radius 1812 testing123

# Common issue: NAS-IPv6-Address not matching client block
# FreeRADIUS looks up client by source IP of RADIUS packet
# NAS-IPv6-Address attribute is informational — not used for auth
```

## Conclusion

NAS-IPv6-Address (attribute 95) is the IPv6 equivalent of NAS-IP-Address for identifying RADIUS clients. Configure it on Cisco IOS by setting `ip radius source-interface` to an IPv6 loopback, and on Juniper by setting `source-address` in the RADIUS server configuration. FreeRADIUS receives this attribute but authenticates the NAS by its source IP address (matched against `clients.conf`). Use `NAS-IPv6-Address` in FreeRADIUS policy (`unlang`) to apply site-specific configurations, and log it in SQL accounting to track which NAS is handling each session.
