# How to Configure IPv6 for Wi-Fi Guest Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Wi-Fi, Guest Network, SSID, VLAN, Isolation, Firewall

Description: Configure IPv6 for Wi-Fi guest networks with proper VLAN isolation, separate prefix delegation, guest-specific firewall policies, and DHCPv6 to prevent access to internal IPv6 resources.

---

Guest Wi-Fi networks need IPv6 internet access while being isolated from the corporate network. Each guest SSID should use a separate IPv6 prefix on a dedicated VLAN, with firewall rules preventing access to RFC 1918 and internal IPv6 ranges.

## Guest Network Architecture

```
IPv6 Guest Network Architecture:
Internet (2001:db8::/32 from ISP)
         |
    [Router/Firewall]
    /               \
[Corp VLAN 10]    [Guest VLAN 20]
2001:db8:corp::/64  2001:db8:guest::/64
Corp Wi-Fi SSID     Guest Wi-Fi SSID
    |                    |
[Corp clients]       [Guest clients]
   Firewall: full     Firewall: internet only,
   internal access    block internal
```

## radvd Configuration - Guest SSID

```bash
# /etc/radvd.conf - Separate RA per SSID/VLAN

# Corporate network (VLAN 10)
interface vlan10 {
    AdvSendAdvert on;
    MinRtrAdvInterval 10;
    MaxRtrAdvInterval 30;

    RDNSS 2001:db8:corp::dns {
        AdvRDNSSLifetime 3600;
    };

    prefix 2001:db8:corp::/64 {
        AdvOnLink on;
        AdvAutonomous on;
    };
};

# Guest network (VLAN 20)
interface vlan20 {
    AdvSendAdvert on;
    MinRtrAdvInterval 10;
    MaxRtrAdvInterval 30;

    # Use public DNS for guests
    RDNSS 2606:4700:4700::1111 2001:4860:4860::8888 {
        AdvRDNSSLifetime 3600;
    };

    prefix 2001:db8:guest::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvValidLifetime 3600;      # Shorter lease for guests
        AdvPreferredLifetime 1800;
    };
};
```

## ip6tables Guest Isolation Rules

```bash
#!/bin/bash
# ipv6-guest-isolation.sh

CORP_VLAN="vlan10"
GUEST_VLAN="vlan20"
WAN_IFACE="eth0"

CORP_PREFIX="2001:db8:corp::/64"
GUEST_PREFIX="2001:db8:guest::/64"
INTERNAL_RANGES="2001:db8::/32 fd00::/8 fc00::/7"

# Allow established connections
ip6tables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow ICMPv6 everywhere (required for NDP/SLAAC)
ip6tables -A FORWARD -p icmpv6 -j ACCEPT

# Guest -> Internet: ALLOW
ip6tables -A FORWARD -i $GUEST_VLAN -o $WAN_IFACE -j ACCEPT

# Guest -> Corp: DENY (IPv6 internal prefixes)
for RANGE in $INTERNAL_RANGES; do
    ip6tables -A FORWARD -i $GUEST_VLAN -d $RANGE -j DROP
done

# Guest -> Guest: DENY (client isolation)
ip6tables -A FORWARD -i $GUEST_VLAN -o $GUEST_VLAN -j DROP

# Corp -> anywhere: ALLOW (with specific rules as needed)
ip6tables -A FORWARD -i $CORP_VLAN -j ACCEPT

# Save rules
ip6tables-save > /etc/ip6tables/rules.v6
echo "Guest IPv6 isolation rules applied"
```

## DHCPv6 for Guest Network (ISC DHCP)

```bash
# /etc/dhcp/dhcpd6.conf

# Corporate network - stateful DHCPv6 with internal DNS
subnet6 2001:db8:corp::/64 {
    range6 2001:db8:corp::100 2001:db8:corp::500;
    option dhcp6.domain-search "corp.example.com";
    option dhcp6.name-servers 2001:db8:corp::dns;
    default-lease-time 86400;
    max-lease-time 172800;
}

# Guest network - stateful DHCPv6 with public DNS
subnet6 2001:db8:guest::/64 {
    range6 2001:db8:guest::100 2001:db8:guest::500;
    option dhcp6.domain-search "guest.example.com";
    # Use public DNS, not internal
    option dhcp6.name-servers 2606:4700:4700::1111 2001:4860:4860::8888;
    # Shorter lease for guests
    default-lease-time 3600;
    max-lease-time 7200;
}
```

## nftables Guest Isolation (Modern Approach)

```bash
# /etc/nftables-guest.conf

table ip6 guest_isolation {

    chain forward {
        type filter hook forward priority filter; policy accept;

        # Allow established/related
        ct state established,related accept

        # Allow ICMPv6
        meta l4proto icmpv6 accept

        # Guest -> Internal corporate: DROP
        iifname "vlan20" ip6 daddr { 2001:db8:corp::/64, fd00::/8 } \
            log prefix "GUEST-BLOCK: " drop

        # Guest -> Guest (client isolation): DROP
        iifname "vlan20" oifname "vlan20" drop

        # Guest -> Internet: ALLOW
        iifname "vlan20" oifname "eth0" accept

        # Log and drop unmatched guest traffic
        iifname "vlan20" log prefix "GUEST-DROP: " drop
    }
}
```

## Verify Guest IPv6 Isolation

```bash
# From a guest wireless client, verify:
# 1. Gets global IPv6 address in guest prefix
ip -6 addr show wlan0 | grep "2001:db8:guest"

# 2. Can reach internet
ping6 2606:4700:4700::1111

# 3. Cannot reach corporate resources
ping6 2001:db8:corp::10  # Should be unreachable

# 4. Cannot reach other guest clients
ping6 2001:db8:guest::101  # Should be blocked

# From the router/firewall, monitor guest traffic
sudo ip6tables -L FORWARD -n -v | grep -E "DROP|guest"

# Check guest DHCPv6 leases
grep "2001:db8:guest" /var/lib/dhcpd/dhcpd6.leases
```

Guest IPv6 networks require a dedicated prefix separate from corporate ranges, with nftables or ip6tables rules blocking forwarding from the guest VLAN to any internal IPv6 prefixes while permitting outbound internet traffic, and DNSSL/RDNSS options in RA pointing guests to public resolvers rather than internal DNS servers.
