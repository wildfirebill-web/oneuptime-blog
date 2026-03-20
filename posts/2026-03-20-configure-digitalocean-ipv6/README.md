# How to Configure DigitalOcean Droplets with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DigitalOcean, Droplets, Cloud, Networking

Description: Configure IPv6 on DigitalOcean Droplets, including enabling IPv6 at creation, configuring the OS, and setting up firewall rules for IPv6 traffic.

## Introduction

DigitalOcean provides native IPv6 for Droplets at no additional cost. Each Droplet receives a public IPv6 /124 allocation. Enabling IPv6 provides a globally routable IPv6 address alongside the IPv4 address.

## Enabling IPv6 at Droplet Creation

```bash
# Using doctl CLI

doctl compute droplet create my-server \
  --region nyc3 \
  --size s-2vcpu-4gb \
  --image ubuntu-22-04-x64 \
  --ipv6 \
  --ssh-keys "your-key-id"

# Verify IPv6 is assigned
doctl compute droplet get my-server --format IPv6
```

## Enabling IPv6 via Terraform

```hcl
# main.tf
resource "digitalocean_droplet" "web" {
  name   = "web-server"
  size   = "s-2vcpu-4gb"
  image  = "ubuntu-22-04-x64"
  region = "nyc3"
  ipv6   = true
  ssh_keys = [var.ssh_key_id]
}

output "ipv6_address" {
  value = digitalocean_droplet.web.ipv6_address
}
```

## Configuring the OS for IPv6

```bash
# On Ubuntu 22.04 - DigitalOcean auto-configures IPv6 via cloud-init
# Verify the address is assigned
ip -6 addr show eth0
# inet6 2604:a880:400:d0::1/64 scope global

# Verify default IPv6 route
ip -6 route show default
# default via 2604:a880:400:d0::1 dev eth0

# If IPv6 is not configured after enabling, add manually
# /etc/netplan/50-cloud-init.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - "2604:a880:400:d0::1/124"
      routes:
        - to: ::/0
          via: "2604:a880:400:d0::1"
          on-link: true
      nameservers:
        addresses:
          - "2001:4860:4860::8888"
          - "2606:4700:4700::1111"

sudo netplan apply
```

## DigitalOcean Firewall for IPv6

```bash
# Create firewall rules for IPv6
doctl compute firewall create \
  --name my-firewall \
  --inbound-rules "protocol:tcp,ports:80,address:0.0.0.0/0,address:::/0" \
  --inbound-rules "protocol:tcp,ports:443,address:0.0.0.0/0,address:::/0" \
  --inbound-rules "protocol:tcp,ports:22,address:your-ip/32,address:your-ipv6/128" \
  --outbound-rules "protocol:tcp,ports:all,address:0.0.0.0/0,address:::/0" \
  --outbound-rules "protocol:udp,ports:all,address:0.0.0.0/0,address:::/0" \
  --outbound-rules "protocol:icmp,address:0.0.0.0/0,address:::/0"

# Assign firewall to droplet
doctl compute firewall add-droplets my-firewall \
  --droplet-ids $(doctl compute droplet get my-server --format ID --no-header)
```

## Testing IPv6 Connectivity

```bash
# From the Droplet
ping6 2001:4860:4860::8888
curl -6 https://ipv6.icanhazip.com

# Verify IPv6 DNS resolution
dig AAAA example.com @2001:4860:4860::8888

# From outside: test that Droplet is reachable via IPv6
ping6 2604:a880:400:d0::1
```

## NGINX Dual-Stack Configuration

```nginx
# /etc/nginx/sites-available/default
server {
    listen 80;
    listen [::]:80;      # IPv6
    listen 443 ssl;
    listen [::]:443 ssl; # IPv6 HTTPS

    server_name example.com;
    # ... rest of config
}
```

## Conclusion

DigitalOcean makes IPv6 configuration straightforward with the `--ipv6` flag or Terraform `ipv6 = true`. Ubuntu cloud images auto-configure via cloud-init. Apply DigitalOcean Cloud Firewalls with `::/0` rules for IPv6 ingress control. Monitor your Droplet's IPv6 availability and response time with OneUptime uptime monitoring.
