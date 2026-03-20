# How to Self-Host an Ad Blocker (DNS) with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Self-Hosted, Pi-hole, AdGuard Home, DNS, Ad Blocking, Networking

Description: Deploy Pi-hole or AdGuard Home as a network-wide DNS-based ad blocker using Portainer to block ads on all devices.

## Introduction

A DNS-based ad blocker filters advertisement and tracking domains at the network level, blocking ads on every device including smart TVs, phones, and IoT devices — without installing any browser extension. Pi-hole and AdGuard Home are the two most popular solutions. This guide covers deploying both using Portainer.

## Prerequisites

- Portainer installed and running
- A static IP for your Docker host
- Access to your router settings to change DNS

## How DNS-Based Ad Blocking Works

When a device requests `ads.doubleclick.net`, your DNS server (Pi-hole/AdGuard) recognizes it as an ad domain and returns `0.0.0.0` instead of the real IP, effectively blocking the request before it reaches your browser.

## Option 1: Deploy Pi-hole

```yaml
# docker-compose.yml - Pi-hole
version: "3.8"

networks:
  dns_network:
    driver: bridge
    ipam:
      config:
        # Fixed subnet for predictable IPs
        - subnet: 172.21.0.0/24

volumes:
  pihole_etc:
  pihole_dnsmasq:

services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    restart: unless-stopped
    # Use host networking for DNS (port 53)
    # Or use port mapping as shown below
    ports:
      - "53:53/tcp"   # DNS TCP
      - "53:53/udp"   # DNS UDP
      - "67:67/udp"   # DHCP (optional)
      - "8053:80/tcp" # Web admin UI
    environment:
      # Admin password for web UI
      - WEBPASSWORD=your_secure_admin_password

      # Upstream DNS servers
      - PIHOLE_DNS_=1.1.1.1;1.0.0.2;9.9.9.9

      # Your local domain (optional)
      - VIRTUAL_HOST=pihole.local

      # Timezone
      - TZ=America/New_York

      # DNSSEC validation
      - DNSSEC=true

      # Enable FTLDNS privacy
      - FTLCONF_PRIVACYLEVEL=0

      # IPv6 support
      - IPv6=false
    volumes:
      # Pi-hole configuration
      - pihole_etc:/etc/pihole
      # DNSMasq configuration
      - pihole_dnsmasq:/etc/dnsmasq.d
    # Required for DNS
    cap_add:
      - NET_ADMIN
    networks:
      dns_network:
        ipv4_address: 172.21.0.100
```

## Option 2: Deploy AdGuard Home

AdGuard Home offers a more modern interface and built-in HTTPS/DoH support.

```yaml
# docker-compose.yml - AdGuard Home
version: "3.8"

networks:
  dns_network:
    driver: bridge

volumes:
  adguard_work:
  adguard_conf:

services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    ports:
      - "53:53/tcp"    # DNS TCP
      - "53:53/udp"    # DNS UDP
      - "3000:3000"    # Initial setup UI
      - "8080:80"      # Web admin UI (after setup)
      - "443:443"      # HTTPS
      - "784:784/udp"  # DNS over QUIC
      - "853:853/tcp"  # DNS over TLS
    volumes:
      # Work directory (query logs, statistics)
      - adguard_work:/opt/adguardhome/work
      # Configuration directory
      - adguard_conf:/opt/adguardhome/conf
    cap_add:
      - NET_ADMIN
    networks:
      - dns_network
```

## Step 3: Configure Router DNS

Point all devices on your network to use Pi-hole/AdGuard as the DNS server.

### Option A: Router DHCP Settings
1. Log into your router admin panel
2. Find **DHCP Settings** or **LAN Settings**
3. Set **Primary DNS** to your Docker host IP (e.g., `192.168.1.100`)
4. Optionally set **Secondary DNS** to `1.1.1.1` as fallback

### Option B: Individual Device Configuration
```bash
# Linux - modify /etc/resolv.conf
echo "nameserver 192.168.1.100" | sudo tee /etc/resolv.conf

# Or via NetworkManager
nmcli con modify "your-connection" ipv4.dns "192.168.1.100"
nmcli con up "your-connection"
```

## Step 4: Add Custom Blocklists to Pi-hole

```bash
# Add blocklists via Pi-hole API
# Navigate to Settings > Blocklists and add:
# https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
# https://raw.githubusercontent.com/FadeMind/hosts.extras/master/add.Spam/hosts
# https://www.github.developerdan.com/hosts/lists/ads-and-tracking-extended.txt

# Or via command line
docker exec pihole pihole -g  # Update gravity (download blocklists)
```

## Step 5: Configure Local DNS Records

Add your self-hosted services as local DNS records:

```bash
# In Pi-hole: Settings > Local DNS > DNS Records
# Or add to dnsmasq config

cat << 'EOF' > /opt/pihole/dnsmasq/custom.conf
# Local service entries
address=/portainer.home/192.168.1.100
address=/nextcloud.home/192.168.1.100
address=/jellyfin.home/192.168.1.100
address=/grafana.home/192.168.1.100
EOF

# Restart Pi-hole to apply
docker restart pihole
```

## Step 6: Set Up Unbound as Recursive DNS

For maximum privacy, use Unbound as a recursive DNS resolver:

```yaml
# Add to docker-compose.yml
  unbound:
    image: mvance/unbound:latest
    container_name: unbound
    restart: unless-stopped
    volumes:
      - /opt/unbound:/opt/unbound/etc/unbound
    networks:
      dns_network:
        ipv4_address: 172.21.0.101
```

```yaml
# Update Pi-hole to use Unbound
environment:
  - PIHOLE_DNS_=172.21.0.101#5335
```

## Monitoring in Portainer

Use Portainer to monitor your DNS server:
- **Logs**: Check for query errors or blocklist update failures
- **Stats**: Monitor CPU and memory (Pi-hole is very lightweight)
- **Restart**: Quickly restart after configuration changes

```bash
# Check Pi-hole statistics
docker exec pihole pihole status
docker exec pihole pihole -c  # Chronometer (real-time stats)

# Check query logs
docker exec pihole pihole -t  # Tail live queries

# Update blocklists
docker exec pihole pihole -g
```

## Allowlisting Domains

```bash
# Allow a domain that's being incorrectly blocked
docker exec pihole pihole -w example.com

# Allow a regex pattern
docker exec pihole pihole --white-regex '^ads\..*\.yoursite\.com$'
```

## Conclusion

You now have a network-wide ad blocker running in Docker managed through Portainer. Every device on your network — phones, smart TVs, gaming consoles, and laptops — benefits from ad and tracker blocking without any configuration on individual devices. Pi-hole typically blocks 15-25% of all DNS queries in a household, significantly reducing tracking and improving page load times. Use Portainer to keep your ad blocker updated and monitor its resource usage.
