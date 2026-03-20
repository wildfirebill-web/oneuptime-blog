# How to Set Up a Transparent Squid Proxy for IPv4 on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Squid, Transparent Proxy, IPv4, iptables, Linux, HTTP Interception

Description: Configure Squid as a transparent HTTP proxy that intercepts IPv4 traffic at the network level using iptables REDIRECT rules, without requiring client configuration.

## Introduction

A transparent proxy intercepts HTTP traffic without clients needing to configure proxy settings. Traffic is redirected at the iptables level to Squid's interception port. HTTPS (CONNECT) cannot be transparently proxied without SSL bump (which involves man-in-the-middle certificate interception).

## Architecture

```
Client (10.0.1.x)
    ↓ HTTP request to 0.0.0.0:80
Linux Router/Gateway (10.0.1.1)
    ↓ iptables REDIRECT to 3127
Squid (transparent port 3127)
    ↓
Destination Web Server
```

## Step 1: Configure Squid for Transparent Interception

```
# /etc/squid/squid.conf

# Standard proxy port
http_port 3128

# Transparent interception port
http_port 3127 intercept

# ACLs
acl localnet src 10.0.1.0/24
acl Safe_ports port 80 443 21 8080
acl CONNECT method CONNECT

http_access deny !Safe_ports
http_access allow localnet
http_access deny all

access_log /var/log/squid/access.log squid
```

## Step 2: Enable IP Forwarding

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Step 3: Configure iptables to Redirect HTTP to Squid

```bash
# Redirect HTTP traffic (port 80) from local subnet to Squid on port 3127
sudo iptables -t nat -A PREROUTING \
  -i eth1 \
  -s 10.0.1.0/24 \
  -p tcp \
  --dport 80 \
  -j REDIRECT --to-port 3127

# Do not redirect Squid's own traffic (avoid loops)
sudo iptables -t nat -A PREROUTING \
  -i eth1 \
  -s 10.0.1.0/24 \
  -p tcp \
  --dport 80 \
  -m owner --uid-owner proxy \
  -j ACCEPT

# Allow forwarding
sudo iptables -A FORWARD -i eth1 -o eth0 -s 10.0.1.0/24 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -d 10.0.1.0/24 -j ACCEPT

# NAT outbound traffic to the internet
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

## Step 4: Save iptables Rules

```bash
sudo apt-get install -y iptables-persistent
sudo netfilter-persistent save
```

## Step 5: Restart Squid

```bash
sudo systemctl restart squid
sudo systemctl enable squid
```

## Verifying Transparent Proxy

From a client on the 10.0.1.0/24 subnet (with default gateway pointing to 10.0.1.1):

```bash
# No proxy settings configured — traffic goes through gateway
curl http://example.com

# Check Squid access log on the gateway
sudo tail -f /var/log/squid/access.log
```

The access log should show the client's request even though no proxy was configured.

## Handling HTTPS Transparently (SSL Bump)

HTTPS cannot be truly transparently proxied without SSL interception. For HTTPS you have two options:

1. **Pass through**: Allow HTTPS to bypass Squid (most common)
2. **SSL Bump**: Intercept and decrypt (requires CA certificate on all clients)

```bash
# Allow HTTPS to bypass the redirect
sudo iptables -t nat -A PREROUTING \
  -i eth1 \
  -p tcp \
  --dport 443 \
  -j ACCEPT
```

## Checking Squid Cache Manager

```bash
sudo squidclient -h 127.0.0.1 mgr:info 2>/dev/null | grep "Number of clients"
```

## Logging All Intercepted Requests

```bash
sudo tail -f /var/log/squid/access.log | awk '{print $3, $7}'
```

## Conclusion

Transparent Squid proxy requires two things: an `http_port ... intercept` directive in Squid and iptables `PREROUTING REDIRECT` rules to intercept traffic. The Linux gateway must have IP forwarding enabled. HTTPS traffic cannot be transparently proxied without SSL bump. This is commonly deployed on home/office gateways to enforce content policies without client configuration.
