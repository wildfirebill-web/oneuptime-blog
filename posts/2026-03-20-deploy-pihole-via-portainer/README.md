# How to Deploy Pi-hole via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Pi-hole, DNS, Ad Blocking, Docker, Networking, Self-Hosting

Description: Learn how to deploy Pi-hole, the network-wide ad blocker, via Portainer with proper DNS port configuration and a persistent data volume.

---

Pi-hole acts as a DNS sinkhole for your entire network, blocking ads and trackers before they reach any device. Running it in Docker via Portainer makes it easy to update, back up, and restart without touching the host network configuration.

## Prerequisites

- Portainer running on a Linux host
- Port `53` (DNS) available (stop `systemd-resolved` if it occupies port 53)
- A static IP for the Pi-hole container

## Freeing Port 53 on Ubuntu/Debian

Ubuntu uses `systemd-resolved` which binds to port 53. Disable the stub listener first:

```bash
# Edit resolved config to disable the stub DNS listener

sudo sed -i 's/#DNSStubListener=yes/DNSStubListener=no/' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
```

## Compose Stack

```yaml
version: "3.8"

services:
  pihole:
    image: pihole/pihole:latest
    restart: unless-stopped
    ports:
      - "53:53/tcp"      # DNS over TCP
      - "53:53/udp"      # DNS over UDP
      - "8053:80/tcp"    # Pi-hole admin UI
    environment:
      TZ: America/New_York
      WEBPASSWORD: yourpassword    # Change this - admin UI password
      PIHOLE_DNS_: 1.1.1.1;8.8.8.8  # Upstream DNS servers
      DNSMASQ_LISTENING: all
    volumes:
      - pihole_data:/etc/pihole
      - dnsmasq_data:/etc/dnsmasq.d
    # Run with host network mode for best DNS compatibility
    # Alternatively use port mappings as shown above
    dns:
      - 127.0.0.1
      - 1.1.1.1

volumes:
  pihole_data:
  dnsmasq_data:
```

## Deploying

1. In Portainer go to **Stacks > Add Stack**.
2. Name it `pihole`.
3. Set `WEBPASSWORD` and your `TZ`.
4. Click **Deploy the stack**.

Access the admin interface at `http://<host>:8053/admin`.

## Pointing Devices to Pi-hole

On your router, set the primary DNS server to the Pi-hole host IP. All devices on your network will automatically use Pi-hole for DNS resolution. You can also configure individual devices manually.

## Adding Block Lists

In the Pi-hole admin UI go to **Group Management > Adlists** and add community block lists:

```text
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
https://adaway.org/hosts.txt
```

Run **Tools > Update Gravity** after adding lists.

## Monitoring

Use OneUptime to monitor `http://<host>:8053/admin/api.php`. Pi-hole returns a JSON object with query statistics. A non-200 response or empty body means the admin interface is down. Also set a DNS monitor to verify that Pi-hole is resolving queries correctly.
