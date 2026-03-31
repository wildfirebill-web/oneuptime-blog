# How to Run Pi-hole in a Podman Container

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Pi-hole, DNS, Ad Blocking, Network Security

Description: Learn how to run Pi-hole in a Podman container for network-wide DNS-based ad blocking with a web dashboard and custom blocklists.

---

> Pi-hole in Podman provides network-wide ad blocking through DNS filtering in a rootless container with a powerful web dashboard.

Pi-hole is a network-level ad blocker that acts as a DNS sinkhole, filtering out advertisement and tracking domains for all devices on your network. Running it in a Podman container gives you DNS-based ad blocking without dedicated hardware, with easy configuration through a web interface. This guide covers setup, custom DNS configuration, blocklist management, and the web dashboard.

---

## Pulling the Pi-hole Image

Download the official Pi-hole image.

```bash
# Pull the latest Pi-hole image

podman pull docker.io/pihole/pihole:latest

# Verify the image
podman images | grep pihole
```

## Running a Basic Pi-hole Container

Start Pi-hole with DNS and the web interface.

```bash
# Create volumes for Pi-hole configuration and DNS records
podman volume create pihole-config
podman volume create pihole-dnsmasq

# Run Pi-hole with required settings
podman run -d \
  --name my-pihole \
  -p 53:53/tcp \
  -p 53:53/udp \
  -p 8080:80 \
  -e TZ=America/New_York \
  -e WEBPASSWORD=my-pihole-password \
  -v pihole-config:/etc/pihole:Z \
  -v pihole-dnsmasq:/etc/dnsmasq.d:Z \
  pihole/pihole:latest

# Wait for Pi-hole to initialize
sleep 15

# Check the container is running
podman ps

# Verify DNS is working
dig @localhost example.com +short

# Access the web dashboard
echo "Open http://localhost:8080/admin in your browser"
echo "Password: my-pihole-password"
```

## Configuring Upstream DNS Servers

Set custom upstream DNS servers for Pi-hole to forward queries to.

```bash
# Run Pi-hole with custom upstream DNS servers
podman run -d \
  --name pihole-custom-dns \
  -p 5353:53/tcp \
  -p 5353:53/udp \
  -p 8081:80 \
  -e TZ=America/New_York \
  -e WEBPASSWORD=my-pihole-password \
  -e PIHOLE_DNS_="1.1.1.1;1.0.0.1" \
  -e DNSSEC=true \
  -e REV_SERVER=true \
  -e REV_SERVER_TARGET=192.168.1.1 \
  -e REV_SERVER_CIDR=192.168.1.0/24 \
  -v pihole-config:/etc/pihole:Z \
  -v pihole-dnsmasq:/etc/dnsmasq.d:Z \
  pihole/pihole:latest
```

## Adding Custom DNS Records

Define local DNS entries for your network.

```bash
# Add custom DNS records via the configuration file
podman exec my-pihole bash -c 'cat >> /etc/pihole/custom.list <<EOF
192.168.1.10 server.local
192.168.1.20 nas.local
192.168.1.30 printer.local
192.168.1.100 homelab.local
EOF'

# Restart DNS to apply changes
podman exec my-pihole pihole restartdns

# Test the custom DNS record
dig @localhost server.local +short
```

## Managing Blocklists

Add and manage ad-blocking lists.

```bash
# View current blocklist statistics
podman exec my-pihole pihole -c -e

# Update the gravity database (blocklists)
podman exec my-pihole pihole -g

# Add a custom blocklist via the command line
podman exec my-pihole bash -c 'sqlite3 /etc/pihole/gravity.db \
  "INSERT INTO adlist (address, enabled) VALUES (\"https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts\", 1);"'

# Update gravity after adding new lists
podman exec my-pihole pihole -g

# Check how many domains are blocked
podman exec my-pihole pihole -c -e | head -5
```

## Whitelisting and Blacklisting Domains

Control which domains are allowed or blocked.

```bash
# Whitelist a domain
podman exec my-pihole pihole -w example.com

# Whitelist with a comment
podman exec my-pihole pihole -w safe-site.com --comment "Needed for work"

# Blacklist a specific domain
podman exec my-pihole pihole -b tracking.example.com

# Blacklist with a wildcard
podman exec my-pihole pihole --wild-block ads.example.com

# Show the current whitelist
podman exec my-pihole pihole -w -l

# Show the current blacklist
podman exec my-pihole pihole -b -l

# Remove a domain from the whitelist
podman exec my-pihole pihole -w -d example.com
```

## Custom dnsmasq Configuration

Add custom dnsmasq settings for advanced DNS control.

```bash
# Create a custom dnsmasq configuration
podman exec my-pihole bash -c 'cat > /etc/dnsmasq.d/99-custom.conf <<EOF
# Set a custom domain for local network
local=/home.lab/
domain=home.lab

# DHCP range (if Pi-hole should act as DHCP)
# dhcp-range=192.168.1.100,192.168.1.200,24h

# Conditional forwarding for a specific domain
server=/corp.example.com/10.0.0.1

# Cache settings
cache-size=10000
EOF'

# Restart DNS to apply changes
podman exec my-pihole pihole restartdns
```

## Monitoring Pi-hole

Check Pi-hole statistics and query logs.

```bash
# View Pi-hole summary statistics
podman exec my-pihole pihole -c -e

# View the query log (last 20 entries)
podman exec my-pihole pihole -t 20

# Check Pi-hole status
podman exec my-pihole pihole status

# Use the Pi-hole API for stats
curl -s "http://localhost:8080/admin/api.php?summary" | python3 -m json.tool

# Get top blocked domains
curl -s "http://localhost:8080/admin/api.php?topItems=10&auth=$(podman exec my-pihole pihole -a -p my-pihole-password 2>&1 | grep -oP 'New password.*' || echo '')" | python3 -m json.tool

# Temporarily disable Pi-hole blocking (for 5 minutes)
podman exec my-pihole pihole disable 300

# Re-enable Pi-hole blocking
podman exec my-pihole pihole enable
```

## Managing the Container

Common management operations.

```bash
# View Pi-hole logs
podman logs my-pihole

# View DNS query logs
podman exec my-pihole tail -f /var/log/pihole/pihole.log

# Restart the Pi-hole DNS service
podman exec my-pihole pihole restartdns

# Stop and start
podman stop my-pihole
podman start my-pihole

# Remove containers and volumes
podman rm -f my-pihole pihole-custom-dns
podman volume rm pihole-config pihole-dnsmasq
```

## Summary

Running Pi-hole in a Podman container gives you network-wide ad blocking through DNS filtering without dedicated hardware. The web dashboard provides real-time statistics on blocked queries, top domains, and client activity. Custom DNS records let you set up local name resolution, while blocklist management and whitelisting give you granular control over what is filtered. Named volumes persist your configuration and blocklists across restarts. Podman's rootless mode provides security isolation for your DNS infrastructure, though you may need to configure port forwarding for port 53 depending on your system setup.
