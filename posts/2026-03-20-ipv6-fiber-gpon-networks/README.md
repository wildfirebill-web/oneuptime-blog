# How to Configure IPv6 for Fiber (GPON) Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, GPON, Fiber, OLT, ONT, ISP, Broadband

Description: Configure IPv6 for GPON and XGS-PON fiber networks including OLT configuration, subscriber provisioning with DHCPv6-PD, and ONT management.

## GPON IPv6 Architecture

```text
OLT (Optical Line Terminal) - head-end at CO/hub
  ↓ (fiber splitter 1:32 or 1:64)
ONT/ONU (Optical Network Terminal) - at subscriber premises
  ↓
CPE Router (Home Router)
  ↓
Home Devices (SLAAC)
```

IPv6 flows through:
1. OLT bridges Ethernet frames per service (management, internet, IPTV)
2. ONT terminates optical signal, bridges Ethernet to CPE
3. CPE gets DHCPv6 WAN address + /56 prefix delegation from OLT's DHCP relay
4. Home devices get /64 via SLAAC from CPE

## OLT IPv6 Configuration (Huawei MA5800)

```text
# Huawei OLT MA5800 - IPv6 for GPON

# Management IPv6

interface MEth0/0/0
  ipv6 address 2001:db8:mgmt::olt1/64

# Configure VLAN for internet service
vlan 100 smart
  description Internet_Service

# DHCPv6 relay for subscriber VLANs
dhcp-server ipv6 2001:db8::dhcp vrf MANAGEMENT
interface Vlanif100
  ipv6 address 2001:db8:gpon::1/64
  ipv6 dhcp relay destination 2001:db8::dhcp

# RA configuration for subscriber subnets
interface Vlanif100
  ipv6 nd ra-prefix-interval 60
  ipv6 nd prefix 2001:db8:gpon::/64 14400 7200
```

## Nokia ISAM OLT IPv6

```text
# Nokia 7360 ISAM - IPv6 configuration

configure router interface "subscriber"
  ipv6
    address 2001:db8:gpon::1/64
    dhcp6-relay
      server-address 2001:db8::dhcp
      interface "subscriber"
    exit
  exit
```

## DHCPv6 Server for GPON Subscribers

```bash
# Wide DHCPv6 server configuration
# /etc/wide-dhcpv6/dhcp6s.conf

interface pon0 {
    server-preference 255;
    allow rapid-commit;
};

# Address pool for ONT management
pool6 MGMT_POOL {
    range6 2001:db8:mgmt:1000:: 2001:db8:mgmt:1fff::;
};

# Prefix pool for subscriber home networks
pool6 PD_POOL {
    prefix6 2001:db8:home:: 2001:db8:home:ff00:: /56;
};

# Static lease for specific ONT (by DUID)
host ONT_STATIC {
    duid 00:03:00:01:aa:bb:cc:dd:ee:ff;
    prefix6 2001:db8:home:a0::/56;
};
```

## ONT Provisioning Script

```bash
#!/bin/bash
# provision-ipv6.sh - Provision IPv6 for new GPON subscriber

ONT_ID=$1
SUBSCRIBER_ID=$2

# Assign IPv6 /56 from pool
# (In production, this comes from IPAM)
PREFIX=$(python3 -c "
import ipaddress
base = ipaddress.ip_network('2001:db8:home::/40')
subs = list(base.subnets(new_prefix=56))
# Assign based on subscriber ID (simplified)
idx = int('${SUBSCRIBER_ID}') % len(subs)
print(subs[idx])
")

echo "Assigning ${PREFIX} to ONT ${ONT_ID}"

# Update DHCPv6 server with static lease
cat >> /etc/wide-dhcpv6/dhcp6s.conf << EOF
host ONT_${ONT_ID} {
    duid 00:03:00:01:$(get_ont_mac ${ONT_ID});
    prefix6 ${PREFIX};
};
EOF

# Reload DHCPv6 server
systemctl reload wide-dhcpv6-server

echo "Provisioned: ONT ${ONT_ID} → ${PREFIX}"
```

## Monitoring GPON IPv6

```bash
# Check active ONT IPv6 leases
grep "IA_PREFIX" /var/lib/dhcpv6/dhcp6s.leases | \
    awk '{print $2}' | sort | uniq -c | sort -rn | head -10

# ONT registration statistics
ip -6 neigh show dev pon0 | wc -l
echo "Active ONT IPv6 neighbors"

# DHCPv6 lease count
wc -l /var/lib/dhcpv6/dhcp6s.leases
```

## Conclusion

GPON IPv6 provisioning uses DHCPv6-PD to assign /56 prefixes to each CPE router behind the ONT. The OLT acts as a DHCPv6 relay, forwarding subscriber DHCPv6 requests to the central provisioning server. Configure OLT VLANs with `ipv6 dhcp relay destination` pointing to the DHCPv6 server. Each subscriber's CPE router receives a /56 and sub-delegates /64s to home devices via SLAAC. Use DUID-based static leases in the DHCPv6 server to ensure consistent prefix assignment across reboots and power outages.
