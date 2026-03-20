# How to Use Local DNS with Portainer (Pi-hole/AdGuard)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, DNS, Pi-hole, AdGuard, Networking

Description: Configure Pi-hole or AdGuard Home as a local DNS resolver to give Portainer services friendly domain names on your home or office network.

## Introduction

Running Pi-hole or AdGuard Home alongside Portainer lets you access all your services via memorable domain names instead of IP:port combinations. For example, accessing `portainer.home` instead of `192.168.1.50:9443`. This guide shows how to deploy these DNS tools via Portainer and configure local DNS records.

## Deploying Pi-hole via Portainer

```yaml
# Deploy Pi-hole as a Portainer stack
version: '3.8'
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    network_mode: host  # Required for DNS on port 53
    environment:
      TZ: 'America/New_York'
      WEBPASSWORD: 'your-admin-password'
      PIHOLE_DNS_: '1.1.1.1;8.8.8.8'
      DNSMASQ_LISTENING: 'all'
    volumes:
      - pihole-config:/etc/pihole
      - pihole-dnsmasq:/etc/dnsmasq.d
    restart: unless-stopped
    cap_add:
      - NET_ADMIN  # Required for network operations

volumes:
  pihole-config:
  pihole-dnsmasq:
```

## Deploying AdGuard Home via Portainer

```yaml
# AdGuard Home stack
version: '3.8'
services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    network_mode: host
    volumes:
      - adguard-work:/opt/adguardhome/work
      - adguard-conf:/opt/adguardhome/conf
    restart: unless-stopped

volumes:
  adguard-work:
  adguard-conf:
```

## Adding Local DNS Records to Pi-hole

### Via Pi-hole Admin UI

1. Access Pi-hole admin: `http://pihole-ip/admin`
2. Go to **Local DNS > DNS Records**
3. Add records:

| Domain | IP |
|--------|----|
| portainer.home | 192.168.1.50 |
| app.home | 192.168.1.51 |
| db.home | 192.168.1.52 |
| grafana.home | 192.168.1.53 |

### Via Pi-hole CLI

```bash
# SSH into the Pi-hole container
docker exec -it pihole bash

# Add custom DNS records
echo "address=/portainer.home/192.168.1.50" >> /etc/dnsmasq.d/custom-dns.conf
echo "address=/app.home/192.168.1.51" >> /etc/dnsmasq.d/custom-dns.conf
echo "address=/*.home/192.168.1.50" >> /etc/dnsmasq.d/custom-wildcard.conf

# Restart dnsmasq
pihole restartdns
```

### Via AdGuard Home

```bash
# AdGuard Home stores DNS rewrites in its config
# Access API to add rewrites
curl -X POST \
  http://adguard-host:3000/control/rewrite/add \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "Content-Type: application/json" \
  -d '{"domain": "portainer.home", "answer": "192.168.1.50"}'

# Add wildcard for all .home domains
curl -X POST \
  http://adguard-host:3000/control/rewrite/add \
  -H "Authorization: Basic $(echo -n 'admin:password' | base64)" \
  -H "Content-Type: application/json" \
  -d '{"domain": "*.home", "answer": "192.168.1.50"}'
```

## Configure Devices to Use Local DNS

```bash
# Option 1: Set router's DHCP DNS to point to Pi-hole/AdGuard IP
# Router settings: DHCP > DNS Server: 192.168.1.53

# Option 2: Static DNS on individual devices
# Linux: /etc/resolv.conf
echo "nameserver 192.168.1.53" | sudo tee /etc/resolv.conf

# Or via systemd-resolved
sudo tee /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=192.168.1.53
FallbackDNS=1.1.1.1
Domains=home
EOF
sudo systemctl restart systemd-resolved

# Option 3: Docker daemon DNS
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "dns": ["192.168.1.53"],
  "dns-search": ["home"]
}
EOF
sudo systemctl restart docker
```

## SSL Certificates for Local Domains

For HTTPS on local domains, use self-signed certs or local CA:

```bash
# Create a local CA
openssl genrsa -out local-ca.key 4096
openssl req -x509 -new -nodes -key local-ca.key -sha256 -days 3650 \
  -out local-ca.crt -subj "/CN=Home CA"

# Trust the CA on all devices
# macOS: sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain local-ca.crt
# Linux: sudo cp local-ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates

# Generate cert for portainer.home
openssl req -new -newkey rsa:2048 -nodes \
  -keyout portainer.home.key \
  -out portainer.home.csr \
  -subj "/CN=portainer.home"

openssl x509 -req -in portainer.home.csr \
  -CA local-ca.crt -CAkey local-ca.key -CAcreateserial \
  -out portainer.home.crt -days 365 \
  -extfile <(echo "subjectAltName=DNS:portainer.home,DNS:*.home")
```

## Verifying DNS Resolution

```bash
# Test from your machine
nslookup portainer.home 192.168.1.53
dig portainer.home @192.168.1.53

# Test from inside a Docker container
docker run --rm --dns 192.168.1.53 alpine nslookup portainer.home

# Check Pi-hole query log
docker exec pihole pihole -t | grep portainer.home
```

## Conclusion

Using Pi-hole or AdGuard Home with Portainer gives your home or office network friendly hostnames for all services. The setup is entirely self-contained within Portainer stacks, and adding new service DNS entries takes seconds. Combined with local SSL certificates, you get a professional, HTTPS-secured access experience for all self-hosted services.
