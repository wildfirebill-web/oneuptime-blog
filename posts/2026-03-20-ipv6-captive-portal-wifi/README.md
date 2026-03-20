# How to Configure IPv6 Captive Portals for Wi-Fi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Captive Portal, Wi-Fi, Guest Network, Authentication, Hotspot

Description: Configure IPv6-compatible captive portals for Wi-Fi guest networks, handling IPv6 redirect challenges, DNS-based interception, and dual-stack portal authentication.

---

Captive portals intercept unauthenticated users and redirect them to a login page. IPv6 complicates captive portals because clients may bypass DNS-based redirection using direct IPv6 addresses, and browsers behave differently for IPv6 portal detection. A proper IPv6 captive portal handles both address families.

## IPv6 Captive Portal Challenges

```
IPv6 Captive Portal Issues:
1. Clients may have multiple IPv6 addresses (SLAAC + DHCPv6 + privacy)
2. HTTPS sites show certificate errors when redirected
3. OS captive portal detection uses known IPv6 probe addresses
4. IPv6 traffic may bypass iptables if only IPv4 rules are set

Solutions:
- Use nftables/ip6tables for IPv6 traffic interception
- Block direct IPv6 internet access until authenticated
- Support CAPPORT API (RFC 8910) for smooth portal detection
- Track sessions by MAC address, not IP
```

## nftables IPv6 Captive Portal Rules

```bash
#!/bin/bash
# ipv6-captive-portal.sh - Set up IPv6 captive portal intercept

PORTAL_IP6="2001:db8::portal"
PORTAL_PORT="443"
DNS_IP6="2001:db8::dns"

# Flush existing rules
nft flush ruleset

# Create nftables configuration
cat > /etc/nftables-captive.conf << 'EOF'
table ip6 captive_portal {

    # Authenticated clients set (populated by portal)
    set authenticated {
        type ipv6_addr
        flags interval, timeout
        timeout 8h
    }

    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;

        # Allow authenticated clients through
        ip6 saddr @authenticated accept

        # Allow ICMPv6 (NDP, ping)
        meta l4proto icmpv6 accept

        # Allow DHCPv6
        udp dport 547 accept

        # Allow DNS (redirect to local DNS)
        udp dport 53 redirect to :53
        tcp dport 53 redirect to :53

        # Redirect HTTP to captive portal
        tcp dport 80 redirect to :8080

        # Redirect HTTPS to captive portal (shows cert error - unavoidable)
        tcp dport 443 dnat to [2001:db8::portal]:443
    }

    chain forward {
        type filter hook forward priority filter; policy drop;

        # Allow authenticated clients to forward
        ip6 saddr @authenticated accept

        # Allow established connections
        ct state established,related accept

        # Allow ICMPv6
        meta l4proto icmpv6 accept

        # Block everything else
        drop
    }
}
EOF

nft -f /etc/nftables-captive.conf
echo "IPv6 captive portal rules loaded"
```

## Add Authenticated Client to Allow List

```bash
# When client authenticates, add their IPv6 to the set
# Note: Track by MAC as clients may have multiple IPv6 addresses

# Get all IPv6 addresses for a MAC (from NDP table)
CLIENT_MAC="aa:bb:cc:dd:ee:ff"
CLIENT_IPS=$(ip -6 neigh show | grep "$CLIENT_MAC" | awk '{print $1}')

# Add all IPv6 addresses for this client
for IP in $CLIENT_IPS; do
    nft add element ip6 captive_portal authenticated { $IP }
    echo "Authenticated IPv6: $IP"
done

# Remove on logout/expiry (automatic with timeout flag)
# Or manual removal:
nft delete element ip6 captive_portal authenticated { 2001:db8::client1 }
```

## CAPPORT API for IPv6 (RFC 8910)

```python
#!/usr/bin/env python3
# capport_api.py - CAPPORT API endpoint for IPv6 captive portal

from flask import Flask, request, jsonify, redirect
import socket

app = Flask(__name__)

@app.route('/api/v1/status')
def captive_status():
    """CAPPORT API status endpoint (RFC 8910)."""
    client_ip = request.remote_addr

    # Check if client is authenticated
    is_authenticated = check_auth(client_ip)

    response = {
        "captive": not is_authenticated,
        "user-portal-url": "https://portal.example.com/login",
        "can-extend-session": True
    }

    if is_authenticated:
        response["seconds-remaining"] = get_remaining_seconds(client_ip)
        response.pop("user-portal-url", None)

    return jsonify(response)

def check_auth(ip):
    """Check if IP is in authenticated set."""
    # Query nftables set
    import subprocess
    result = subprocess.run(
        ['nft', 'list', 'set', 'ip6', 'captive_portal', 'authenticated'],
        capture_output=True, text=True
    )
    return ip in result.stdout

def get_remaining_seconds(ip):
    """Return session time remaining."""
    return 3600  # Placeholder

@app.route('/login', methods=['GET', 'POST'])
def login_portal():
    """Handle portal login."""
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        client_ip = request.remote_addr

        if authenticate_user(username, password):
            authorize_client(client_ip)
            return redirect("https://portal.example.com/success")

    return '''<html><body>
    <h2>Wi-Fi Login</h2>
    <form method="POST">
    Username: <input name="username"><br>
    Password: <input type="password" name="password"><br>
    <input type="submit" value="Login">
    </form></body></html>'''

def authenticate_user(username, password):
    return True  # Implement actual auth

def authorize_client(client_ip):
    import subprocess
    subprocess.run(['nft', 'add', 'element', 'ip6', 'captive_portal',
                    'authenticated', '{', client_ip, '}'])

if __name__ == '__main__':
    app.run(host='::', port=443, ssl_context='adhoc')
```

## RA Options for Captive Portal Detection

```bash
# RFC 7710 / RFC 8910: Advertise captive portal URL in RA
# radvd.conf
interface wlan0 {
    AdvSendAdvert on;
    # ...
    prefix 2001:db8:guest::/64 {
        AdvAutonomous on;
        AdvOnLink on;
    };
    # Captive Portal URI option (type 37)
    # Not natively supported in radvd - use patched version or
    # advertise via DHCPv6 option 114
};
```

```bash
# DHCPv6 captive portal option in ISC DHCP
# /etc/dhcp/dhcpd6.conf
option dhcp6.capwap-ac-v6 code 112 = string;

subnet6 2001:db8:guest::/64 {
    range6 2001:db8:guest::100 2001:db8:guest::200;
    option dhcp6.domain-search "guest.example.com";
    # option 114: Captive Portal URI
    option dhcp6.capwap-ac-v6 "https://portal.example.com/api/v1/status";
}
```

IPv6 captive portals require tracking clients by MAC address rather than IP since each device may have multiple IPv6 addresses, using nftables with named sets for efficient allow-listing, and implementing the CAPPORT API (RFC 8910) to enable smooth captive portal detection in modern operating systems that actively probe for portal presence.
