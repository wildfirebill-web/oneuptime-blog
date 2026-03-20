# How to Deploy Pi-hole via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Pi-hole, DNS, Ad Blocking, Self-Hosted, Privacy

Description: Deploy Pi-hole via Portainer as a network-wide DNS ad blocker that eliminates ads, trackers, and malware domains for all devices on your network.

## Introduction

Pi-hole is a network-wide DNS ad blocker. By setting it as the DNS server for your router, every device on your network benefits from ad blocking without installing browser extensions. Deploying via Portainer makes it easy to update blocklists and manage the configuration.

## Deploy as a Stack

```yaml
version: "3.8"

services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    network_mode: host   # Required for DNS on port 53
    environment:
      TZ: America/New_York
      WEBPASSWORD: change_this_password
      PIHOLE_DNS_: 1.1.1.1;1.0.0.1  # Upstream DNS servers
      DNSSEC: "true"                  # Enable DNSSEC validation
      REV_SERVER: "true"              # Enable reverse DNS lookup
      REV_SERVER_DOMAIN: local
      REV_SERVER_TARGET: 192.168.1.1  # Your router IP
      REV_SERVER_CIDR: 192.168.1.0/24
      FTLCONF_LOCAL_IPV4: 192.168.1.100  # Pi-hole server IP
    volumes:
      - pihole_etc:/etc/pihole
      - pihole_dnsmasq:/etc/dnsmasq.d
    restart: unless-stopped
```

Note: `network_mode: host` is required because DNS uses port 53 which needs to bind to the host network directly.

## Bridge Network Alternative

If you cannot use host network mode (e.g., on some NAS devices):

```yaml
version: "3.8"

services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    environment:
      TZ: America/New_York
      WEBPASSWORD: change_this_password
      PIHOLE_DNS_: 1.1.1.1;1.0.0.1
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "80:80"         # Admin UI
    volumes:
      - pihole_etc:/etc/pihole
      - pihole_dnsmasq:/etc/dnsmasq.d
    restart: unless-stopped

volumes:
  pihole_etc:
  pihole_dnsmasq:
```

## Accessing Pi-hole Admin

Navigate to `http://<host>/admin` and log in with your WEBPASSWORD.

## Configure Your Router to Use Pi-hole

In your router's DHCP settings, change the DNS server to Pi-hole's IP (e.g., `192.168.1.100`). This makes all network devices use Pi-hole automatically.

## Adding Custom Blocklists

1. In Pi-hole Admin, navigate to **Lists**
2. Click **Add a new list**
3. Popular blocklist URLs:
   - `https://hosts.someonewhocares.org/hosts/zero/hosts`
   - `https://raw.githubusercontent.com/nicehash/spam-blocklists/main/nicehash.txt`
   - `https://raw.githubusercontent.com/anudeepND/blacklist/master/adservers.txt`

## Custom DNS Entries (Local Hostnames)

```bash
# Add custom DNS entries via Pi-hole CLI

docker exec pihole pihole -a --localrecord=mynas.local 192.168.1.50
docker exec pihole pihole -a --localrecord=portainer.local 192.168.1.100

# Or edit custom.list directly
docker exec pihole tee /etc/pihole/custom.list << 'EOF'
192.168.1.50 mynas.local
192.168.1.100 portainer.local
192.168.1.101 homeassistant.local
EOF

docker exec pihole pihole restartdns
```

## Update Gravity (Blocklists)

```bash
# Update all blocklists
docker exec pihole pihole -g

# Or via admin UI: Gravity > Update Gravity
```

## Whitelist Domains

```bash
# Whitelist a domain
docker exec pihole pihole -w example.com

# Remove from whitelist
docker exec pihole pihole -w -d example.com
```

## Pi-hole with Unbound (Recursive DNS)

For ultimate privacy with local recursive DNS resolution:

```yaml
version: "3.8"

services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    network_mode: host
    environment:
      PIHOLE_DNS_: 127.0.0.1#5335  # Point to Unbound
    volumes:
      - pihole_etc:/etc/pihole
      - pihole_dnsmasq:/etc/dnsmasq.d

  unbound:
    image: mvance/unbound:latest
    container_name: unbound
    network_mode: host
    volumes:
      - unbound_config:/opt/unbound/etc/unbound
    restart: unless-stopped
```

## Conclusion

Pi-hole deployed via Portainer provides network-wide ad and tracker blocking for every device on your network. The web admin interface makes it easy to manage blocklists, view query logs, and whitelist legitimate domains. Combined with Unbound for recursive DNS, you eliminate DNS-based tracking entirely and reduce your dependence on commercial DNS providers.
