# How to Use IPv6 for Home Lab Servers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Home Lab, Linux, Networking, Static Address, Server

Description: Configure IPv6 for home lab servers with static addresses, local DNS, reverse proxy, and remote access without NAT.

## Why IPv6 is Great for Home Labs

IPv6 eliminates NAT for home lab servers, giving each server a globally routable address. This means:
- No port forwarding rules to manage
- Each service can have its own address
- Remote access is simpler and more secure
- Multiple services on the same port across different addresses

## Step 1: Assign Static IPv6 Addresses

For home lab servers, assign static addresses from your delegated prefix:

```bash
# /etc/netplan/01-homelab.yaml (Ubuntu)
network:
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: false          # Disable SLAAC on this interface
      addresses:
        - "2001:db8:home::10/64"   # Primary server
      routes:
        - to: "::/0"
          via: "2001:db8:home::1"  # Your router
          metric: 100
      nameservers:
        addresses:
          - "2001:db8:home::1"
          - "2001:4860:4860::8888"
  version: 2

sudo netplan apply
```

## Step 2: Local DNS for Home Lab

Register each server in local DNS for easy access by name:

```
# /etc/hosts on each server (or dnsmasq config)
2001:db8:home::10  homeserver.lab.home
2001:db8:home::20  nas.lab.home
2001:db8:home::30  pihole.lab.home
2001:db8:home::40  proxmox.lab.home
```

Or configure dnsmasq with AAAA records:

```
# /etc/dnsmasq.d/homelab.conf
aaaa-record=homeserver.lab.home,2001:db8:home::10
aaaa-record=nas.lab.home,2001:db8:home::20
aaaa-record=proxmox.lab.home,2001:db8:home::40
```

## Step 3: Configure SSH for IPv6

SSH works over IPv6 without any changes. Connect using:

```bash
# Connect to home lab server by IPv6 address
ssh user@2001:db8:home::10

# Or by hostname (if DNS is configured)
ssh user@homeserver.lab.home

# If using link-local (no global address):
ssh user@fe80::1234:5678:abcd:ef01%eth0
```

SSH server configuration for IPv6:

```
# /etc/ssh/sshd_config
ListenAddress ::    # Listen on all IPv6 interfaces
ListenAddress 0.0.0.0  # And IPv4
AddressFamily any   # Accept both
```

## Step 4: Web Services Over IPv6

Nginx configuration to serve web content over IPv6:

```nginx
server {
    # Listen on IPv6 (and IPv4 via dual-stack)
    listen [::]:80 ipv6only=off;
    listen [::]:443 ssl ipv6only=off;

    server_name homeserver.lab.home;

    ssl_certificate     /etc/ssl/certs/homelab.crt;
    ssl_certificate_key /etc/ssl/private/homelab.key;

    root /var/www/homelab;
    index index.html;
}
```

Access via browser: `https://[2001:db8:home::10]` (note the square brackets)

## Step 5: Docker with IPv6 in Home Lab

Enable IPv6 in Docker daemon:

```json
// /etc/docker/daemon.json
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:home:docker::/64",
  "ip6tables": true
}
```

Docker compose with IPv6 networking:

```yaml
# docker-compose.yml
version: "3"

networks:
  homelab:
    driver: bridge
    enable_ipv6: true
    ipam:
      config:
        - subnet: "2001:db8:home:docker::/64"

services:
  web:
    image: nginx
    networks:
      homelab:
        ipv6_address: "2001:db8:home:docker::10"
    ports:
      - "[::]:8080:80"
```

## Step 6: Firewall Rules for Home Lab

Apply nftables to control access to home lab servers:

```bash
# /etc/nftables.conf - home lab server firewall

table ip6 filter {
    chain input {
        type filter hook input priority 0; policy drop;

        iif "lo" accept
        ct state established,related accept
        ip6 nexthdr icmpv6 accept

        # SSH only from home LAN
        ip6 saddr 2001:db8:home::/64 tcp dport 22 accept

        # Web services from anywhere
        tcp dport { 80, 443 } accept

        # Prometheus monitoring from monitoring server
        ip6 saddr 2001:db8:home::50 tcp dport 9100 accept
    }
}
```

## Step 7: Remote Access Without Port Forwarding

With IPv6, access your home lab from anywhere on the internet directly:

1. Add firewall rule to allow inbound SSH from your remote IP
2. Connect directly: `ssh user@2001:db8:home::10`

No port forwarding in the router required — the server's IPv6 address is globally routable.

## Conclusion

IPv6 transforms home lab networking by eliminating NAT. Each server gets a globally routable address, enabling direct inbound connections from the internet. Combine static addresses, local DNS, and nftables firewalls for a clean, professional home lab IPv6 setup.
