# How to Configure IPv6 Reverse DNS for ISPs (RFC 8501)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Reverse DNS, ISP, RFC 8501, ip6.arpa

Description: A guide to implementing IPv6 reverse DNS at ISP scale per RFC 8501 guidelines, covering delegation management, customer zone provisioning, and scalability considerations.

## What Is RFC 8501?

RFC 8501, "Reverse DNS in IPv6 for Internet Service Providers," provides operational guidance for ISPs managing reverse DNS for large IPv6 address spaces. It addresses the unique challenges of IPv6 rDNS at carrier scale, where each customer may have a /48, /56, or /64 prefix.

## ISP Reverse DNS Responsibilities

At ISP scale, the ISP must:
1. Manage the parent `ip6.arpa` zone for their assigned /32 block
2. Delegate sub-zones to customers who want to manage their own rDNS
3. Provide PTR records for customers who don't manage their own rDNS
4. Handle the operational scale of potentially millions of addresses

## Setting Up the ISP's Parent Zone

The ISP receives a /32 from their RIR (e.g., ARIN, RIPE). For `2001:db8::/32`:

```named
// ISP's BIND configuration - /etc/named.conf
zone "8.b.d.0.1.0.0.2.ip6.arpa" IN {
    type master;
    file "/var/named/ip6-reverse-master.zone";
    allow-transfer { secondary-ns; };
    // Allow dynamic updates for automated provisioning
    allow-update { key provisioning-tsig; };
};
```

## Customer Delegation Management

RFC 8501 recommends automating customer delegations. When a customer is assigned a prefix, automatically add NS delegation records:

```bash
#!/bin/bash
# Script to add delegation for a new customer prefix
# Usage: ./add-rDNS-delegation.sh 2001:db8:cafe::/48 ns1.customer.com ns2.customer.com

PREFIX=$1
NS1=$2
NS2=$3

# Calculate zone name from prefix
ZONE=$(python3 -c "
import ipaddress
n = ipaddress.ip_network('$PREFIX', strict=False)
full = n.network_address.exploded.replace(':','')
nibbles = list(full[:${#PREFIX}//4*4//4])
nibbles.reverse()
print('.'.join(nibbles) + '.ip6.arpa')
")

echo "Adding delegation for $ZONE to $NS1 and $NS2"

# Add NS records via nsupdate
nsupdate -k /etc/named/provisioning.key << EOF
server 127.0.0.1
zone 8.b.d.0.1.0.0.2.ip6.arpa.
update add $ZONE. 86400 IN NS $NS1.
update add $ZONE. 86400 IN NS $NS2.
send
EOF

echo "Delegation added successfully"
```

## Default PTR Records for Un-Delegated Customers

RFC 8501 recommends that ISPs provide default PTR records for customers who haven't set up their own rDNS. Common patterns:

```dns
; Generic PTR record patterns for un-delegated customer addresses
; Using $GENERATE directive in BIND for automated record creation

; Generate PTR records for all addresses in a /64
; 2001:db8:0:1::/64 → customer1.dynamic.isp.example.com
$GENERATE 0-65535 ${0,4,x}.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa. PTR customer-addr.dynamic.isp.example.com.
```

In practice, ISPs often use a catch-all or wildcard pattern:

```named
// In named.conf: wildcard for non-delegated space
zone "8.b.d.0.1.0.0.2.ip6.arpa" {
    type master;
    file "/var/named/ip6-catch-all.zone";
};
```

```dns
; Wildcard PTR for any undelegated address in the /32
; /var/named/ip6-catch-all.zone
$TTL 3600
@ IN SOA ns1.isp.example.com. noc.isp.example.com. (...)
@ IN NS ns1.isp.example.com.
@ IN NS ns2.isp.example.com.

; Wildcard covers all non-explicitly-delegated addresses
* IN PTR noptr.isp.example.com.
```

## IPAM Integration for Automated PTR Management

Large ISPs use IPAM systems (Infoblox, NetBox, BlueCat) to manage PTR records automatically. When an IPv6 address is assigned to a customer, the IPAM triggers a DNS update:

```python
# Python example: auto-create PTR when assigning IPv6 address
import ipaddress
import subprocess

def create_ptr_record(ipv6_address, hostname):
    """Create a PTR record for an IPv6 address"""
    addr = ipaddress.ip_address(ipv6_address)

    # Generate the PTR record name
    full_hex = addr.exploded.replace(':', '')
    ptr_name = '.'.join(reversed(full_hex)) + '.ip6.arpa.'

    # Add via nsupdate
    update_cmd = f"""
server 127.0.0.1
zone 8.b.d.0.1.0.0.2.ip6.arpa.
update add {ptr_name} 3600 IN PTR {hostname}.
send
"""
    result = subprocess.run(['nsupdate', '-k', '/etc/named/update.key'],
                          input=update_cmd, text=True, capture_output=True)
    return result.returncode == 0
```

## RFC 8501 Key Recommendations Summary

1. **Delegate where possible**: Allow customers to manage their own rDNS
2. **Default records**: Provide generic PTR records for un-delegated space
3. **Automation**: Use dynamic DNS updates (nsupdate, APIs) rather than manual zone file edits
4. **Consistent naming**: Use predictable PTR name formats for automated records
5. **Documentation**: Publish how customers can request rDNS delegation

## Verifying ISP Delegation at Scale

```bash
# Check delegation health for a customer prefix
dig NS e.f.a.c.8.b.d.0.1.0.0.2.ip6.arpa @isp-ns1.example.com

# Bulk check delegations
while read PREFIX; do
    ZONE=$(./calc-zone.sh $PREFIX)
    RESULT=$(dig NS $ZONE @127.0.0.1 +short)
    echo "$PREFIX: $RESULT"
done < customer-prefixes.txt
```

## Summary

RFC 8501 guides ISPs to automate IPv6 rDNS management: maintain a parent ip6.arpa zone, delegate sub-zones to customers via NS records, provide default PTR records for un-delegated space, and integrate with IPAM systems for automated record creation. Automation is essential at ISP scale where manual zone file management is impractical.
