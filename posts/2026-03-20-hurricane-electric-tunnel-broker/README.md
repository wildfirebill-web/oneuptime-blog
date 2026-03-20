# How to Sign Up and Configure Hurricane Electric IPv6 Tunnel Broker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Hurricane Electric, Tunnel Broker, 6in4, Connectivity

Description: Learn how to sign up for Hurricane Electric's free IPv6 tunnel broker, obtain an IPv6 prefix, and configure the 6in4 tunnel on Linux, Cisco, or Windows.

## Overview

Hurricane Electric (HE) at `tunnelbroker.net` is the world's largest IPv6 tunnel broker, operated by one of the most well-connected IPv6 networks. HE provides free 6in4 tunnels with a /64 for the tunnel link and a /48 for routing to your LAN. You can create up to 5 tunnels per account.

## Sign Up

1. Visit https://tunnelbroker.net
2. Click "Register" — create a free account
3. After logging in, click "Create Regular Tunnel"
4. Enter your current IPv4 address (auto-detected or manual)
5. Select a tunnel server (PoP) — choose the closest by latency
6. Click "Create Tunnel"

You'll receive:
```
Server IPv4 Address:   216.218.218.218     (HE's endpoint — example)
Server IPv6 Address:   2001:470:xxxx::1/64 (HE's tunnel interface IP)
Client IPv4 Address:   203.0.113.10        (your WAN IP)
Client IPv6 Address:   2001:470:xxxx::2/64 (your tunnel interface IP)

Routed /48:            2001:470:yyyy::/48  (your LAN prefix)
```

## Linux Configuration

HE provides example config on the tunnel detail page. Click the "Example Configurations" tab and select "Linux-route2":

```bash
# Remove existing sit tunnel if present
sudo ip tunnel del he-ipv6 2>/dev/null

# Create tunnel
sudo ip tunnel add he-ipv6 mode sit \
    remote 216.218.218.218 \
    local  203.0.113.10 \
    ttl 255

sudo ip link set he-ipv6 up mtu 1480

# Assign tunnel endpoint address
sudo ip addr add 2001:470:xxxx::2/64 dev he-ipv6

# Default IPv6 route through tunnel
sudo ip -6 route add ::/0 dev he-ipv6

# Verify
ping6 2001:470:xxxx::1          # Ping HE's tunnel endpoint
ping6 2001:4860:4860::8888      # Ping Google IPv6 DNS
```

## Make Persistent on Linux (systemd-networkd)

```ini
# /etc/systemd/network/10-he-ipv6.netdev
[NetDev]
Name=he-ipv6
Kind=sit

[Tunnel]
Remote=216.218.218.218
Local=203.0.113.10
TTL=255
```

```ini
# /etc/systemd/network/10-he-ipv6.network
[Match]
Name=he-ipv6

[Network]
Address=2001:470:xxxx::2/64

[Route]
Destination=::/0
```

```bash
sudo systemctl restart systemd-networkd
```

## Configure Routed /48 for LAN

Assign a /64 subnet from your /48 to your LAN interface:

```bash
# Use first /64 from your /48 for the LAN
sudo ip addr add 2001:470:yyyy:1::1/64 dev eth1

# Advertise to LAN hosts via radvd
cat > /etc/radvd.conf << 'EOF'
interface eth1 {
    AdvSendAdvert on;
    MinRtrAdvInterval 30;
    MaxRtrAdvInterval 100;

    prefix 2001:470:yyyy:1::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvRouterAddr on;
    };

    RDNSS 2001:470:20::2 { };  # HE's IPv6 DNS (optional)
};
EOF

sudo systemctl enable radvd
sudo systemctl start radvd
```

## Cisco IOS Configuration

From HE's "Example Configurations" tab → Cisco IOS:

```
ipv6 unicast-routing

interface Tunnel0
 no ip address
 ipv6 address 2001:470:xxxx::2/64
 tunnel source GigabitEthernet0/0
 tunnel mode ipv6ip
 tunnel destination 216.218.218.218
 ipv6 mtu 1480

ipv6 route ::/0 Tunnel0

! Distribute /48 to LAN
interface GigabitEthernet0/1
 ipv6 address 2001:470:yyyy:1::1/64
 ipv6 nd ra-interval 30
```

## Windows Configuration

From HE's "Example Configurations" tab → Windows (Vista/7/8/10):

```cmd
netsh interface teredo set state disabled
netsh interface 6to4 set state disabled
netsh interface isatap set state disabled

netsh interface ipv6 add v6v4tunnel IP6Tunnel 203.0.113.10 216.218.218.218
netsh interface ipv6 add address IP6Tunnel 2001:470:xxxx::2
netsh interface ipv6 add route ::/0 IP6Tunnel 2001:470:xxxx::1
```

## Dynamic IPv4 Update

If your ISP assigns dynamic IPv4, update the tunnel endpoint automatically:

```bash
# /etc/dhcp/dhclient-exit-hooks.d/update-he-tunnel
# Replace USERNAME, APIKEY, TUNNELID with your HE account details

if [ "$reason" = "BOUND" ] || [ "$reason" = "RENEW" ] || [ "$reason" = "REBIND" ]; then
    HE_USER="your_username"
    HE_KEY="your_apikey"
    HE_TUNNEL="12345"   # Found in tunnel details URL

    NEW_IPV4="${new_ip_address}"

    # Update HE tunnel with new IPv4
    RESPONSE=$(curl -4 -s \
        "https://${HE_USER}:${HE_KEY}@ipv4.tunnelbroker.net/nic/update?hostname=${HE_TUNNEL}&myip=${NEW_IPV4}")
    logger "HE Tunnel update response: $RESPONSE"

    # Update local tunnel endpoint
    sudo ip tunnel change he-ipv6 local "${NEW_IPV4}"
fi
```

HE API key is found under: My Account → API Key

## Verify Connectivity

```bash
# Ping HE's tunnel endpoint
ping6 -c 3 2001:470:xxxx::1
# PING 2001:470:xxxx::1: 56 data bytes
# 64 bytes from 2001:470:xxxx::1: icmp_seq=1 ttl=64 time=12.5 ms

# Ping an external IPv6 host
ping6 -c 3 2001:4860:4860::8888

# Test browser: visit https://test-ipv6.com
# Should show: "Your IPv6 address: 2001:470:xxxx::2"
# Score: 10/10

# HE IPv6 certification (free): https://ipv6.he.net/certification/
```

## Summary

Hurricane Electric tunnel broker (tunnelbroker.net) provides free 6in4 tunnels with a /64 for the tunnel link and a /48 for your LAN. Sign up, select the nearest PoP, and get the server/client IPv4 and IPv6 addresses. Configure with `ip tunnel add he-ipv6 mode sit remote <HE-IPv4> local <your-IPv4>`. Make persistent with systemd-networkd. Configure radvd to advertise your /48 subnet to LAN hosts. For dynamic IPs, use HE's API (`ipv4.tunnelbroker.net/nic/update`) via dhclient-exit-hooks to keep the tunnel endpoint current.
