# How to Configure IPv6 on Wi-Fi Access Points

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Wi-Fi, Access Point, SLAAC, DHCPv6, Wireless, 802.11

Description: Configure IPv6 on Wi-Fi access points including enabling router advertisement forwarding, SLAAC, DHCPv6 relay, and proper IPv6 prefix delegation for wireless clients.

---

Wi-Fi access points bridge wireless clients to the wired network. IPv6 on access points requires forwarding Router Advertisement (RA) messages from the upstream router to wireless clients, optionally relaying DHCPv6, and ensuring the access point's management interface supports IPv6.

## How IPv6 Works on Wi-Fi Networks

```text
IPv6 Wi-Fi Architecture:
Internet
    |
[Router] 2001:db8::/32 prefix, RA daemon
    |
[Switch]
    |
[Access Point] - bridges L2, forwards RA/DHCPv6
    |
[Wi-Fi Clients] - receive RA, configure SLAAC addresses
  2001:db8::client1/64 (auto-configured)
  2001:db8::client2/64 (auto-configured)
```

## Enable IPv6 Management on Access Point (Generic Linux-Based AP)

```bash
# Many APs run embedded Linux - configure via SSH

# Enable IPv6 on wireless bridge interface

ip -6 addr add 2001:db8::ap1/64 dev br0
ip -6 route add ::/0 via 2001:db8::1 dev br0

# Enable IPv6 forwarding (if AP acts as router)
sysctl -w net.ipv6.conf.all.forwarding=1
sysctl -w net.ipv6.conf.br0.forwarding=1

# Persist across reboots
echo "net.ipv6.conf.all.forwarding=1" >> /etc/sysctl.conf
```

## OpenWrt Access Point IPv6

```bash
# OpenWrt UCI configuration for IPv6

# Enable IPv6 on the LAN bridge
uci set network.lan.ipv6=1
uci set network.lan.ip6assign=64

# Configure DHCPv6 server or SLAAC
uci set dhcp.lan.dhcpv6=server
uci set dhcp.lan.ra=server
uci set dhcp.lan.ra_management=1

# For relay mode (upstream DHCPv6 server)
uci set dhcp.lan.dhcpv6=relay
uci set dhcp.lan.ra=relay
uci set dhcp.wan6.dhcpv6=relay
uci set dhcp.wan6.ra=relay
uci set dhcp.wan6.master=1

uci commit dhcp
uci commit network

# Restart services
/etc/init.d/network restart
/etc/init.d/dnsmasq restart
/etc/init.d/odhcp6c restart

# Verify IPv6 on AP
ip -6 addr show br-lan
ip -6 route show
```

## Verify IPv6 Client Addressing on Wi-Fi

```bash
# Check connected wireless clients have IPv6 (on OpenWrt/hostapd AP)
iw dev wlan0 station dump | grep Station

# Check ARP/NDP table for wireless clients
ip -6 neigh show dev br-lan

# Show all IPv6 addresses given via SLAAC or DHCPv6
# On the AP running odhcpd:
cat /tmp/hosts/odhcpd  # DHCPv6 leases

# On Linux with dnsmasq
cat /var/lib/misc/dnsmasq.leases

# Test IPv6 reachability from AP to a client
ping6 2001:db8::1234 -I br-lan
```

## Enable IPv6 RA Proxy (for Bridged APs)

```bash
# If AP is a pure bridge (no routing), use radvd or ndp-proxy

# On upstream Linux router: enable NDP proxy
sysctl -w net.ipv6.conf.eth0.proxy_ndp=1

# Add NDP proxy entries for wireless clients
ip -6 neigh add proxy 2001:db8::client1 dev eth0

# Or use ndppd for dynamic NDP proxying
# Install ndppd
apt install ndppd -y

# /etc/ndppd.conf
cat > /etc/ndppd.conf << 'EOF'
proxy eth0 {
    rule 2001:db8::/64 {
        auto
    }
}
EOF

systemctl enable --now ndppd
```

## Firewall Rules for Wi-Fi IPv6 Clients

```bash
# Allow ICMPv6 (required for SLAAC/NDP)
sudo ip6tables -A FORWARD -p icmpv6 -j ACCEPT

# Allow DHCPv6 relay
sudo ip6tables -A INPUT -p udp --dport 547 -j ACCEPT
sudo ip6tables -A FORWARD -p udp --dport 546 -j ACCEPT

# Allow Wi-Fi clients to reach internet
sudo ip6tables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
sudo ip6tables -A FORWARD -i eth0 -o wlan0 \
  -m state --state RELATED,ESTABLISHED -j ACCEPT

# Save rules
sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Verify End-to-End IPv6 for Wi-Fi Clients

```bash
# From a connected Wi-Fi client, verify IPv6
ip -6 addr show          # Should show global IPv6 address
ip -6 route show default # Should have default via AP/router
ping6 2606:4700:4700::1111  # Cloudflare DNS over IPv6
curl -6 https://ipv6.google.com  # IPv6 web access

# From the AP, check RA is being sent
radvdump  # Listen for RA messages on the wireless interface

# Verify prefix delegation
dhclient -6 -P -v eth0  # Request prefix from upstream ISP
```

IPv6 on Wi-Fi access points centers on ensuring RA messages propagate correctly from the upstream router to wireless clients via the bridged access point, with NDP proxy or relay mode enabling clients to auto-configure global IPv6 addresses through SLAAC without any additional client configuration.
