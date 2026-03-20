# How to Set vsftpd pasv_address for NAT and IPv4 Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: vsftpd, FTP, Pasv_address, NAT, IPv4, Passive Mode

Description: Configure vsftpd pasv_address to advertise the correct public IPv4 address for passive mode connections through NAT, including dynamic IP support with pasv_addr_resolve.

## Introduction

When vsftpd runs behind NAT, passive mode breaks by default. The server advertises its private IP in the PASV response, but clients cannot reach that address from the internet. The `pasv_address` directive tells vsftpd which IP to advertise-it must be the public IP clients can reach.

## The Problem Without pasv_address

```bash
# Client connects and requests PASV:

# Server responds with private IP (wrong!):
# 227 Entering Passive Mode (10,0,0,5,117,49)
# Client tries to connect to 10.0.0.5:30001 - unreachable from internet!

# With pasv_address set to public IP:
# 227 Entering Passive Mode (203,0,113,10,117,49)
# Client connects to 203.0.113.10:30001 - works!
```

## Setting pasv_address

```bash
# /etc/vsftpd.conf

listen=YES
listen_ipv6=NO

pasv_enable=YES
pasv_min_port=30000
pasv_max_port=31000

# Static public IPv4 address
pasv_address=203.0.113.10
```

## Dynamic IP with pasv_addr_resolve

For servers with dynamic IPs or DNS-based addresses:

```bash
# /etc/vsftpd.conf

pasv_enable=YES
pasv_min_port=30000
pasv_max_port=31000

# Resolve hostname dynamically at connection time
pasv_addr_resolve=YES
pasv_address=ftp.example.com
```

```bash
# DNS A record pointing to your dynamic public IP
# ftp.example.com.  IN A  203.0.113.10

# With a dynamic DNS service (e.g., DuckDNS, No-IP):
# pasv_address=yourdomain.duckdns.org
# pasv_addr_resolve=YES
```

## NAT Gateway Configuration

```bash
# On the NAT gateway, forward FTP traffic to vsftpd server:

# Method 1: iptables DNAT
PUBLIC_IP=203.0.113.10
PRIVATE_IP=10.0.0.5

# Command port
iptables -t nat -A PREROUTING -d $PUBLIC_IP -p tcp --dport 21 \
  -j DNAT --to-destination $PRIVATE_IP:21

# Passive data ports
iptables -t nat -A PREROUTING -d $PUBLIC_IP -p tcp --dport 30000:31000 \
  -j DNAT --to-destination $PRIVATE_IP

# Allow forwarding
iptables -A FORWARD -d $PRIVATE_IP -p tcp -m multiport \
  --dports 21,30000:31000 -j ACCEPT

# Method 2: UFW on gateway
sudo ufw allow proto tcp to $PUBLIC_IP port 21
sudo ufw allow proto tcp to $PUBLIC_IP port 30000:31000
```

## Cloud Provider Configuration

```bash
# AWS Security Group rules for FTP server:
# Inbound:
#   TCP 21       - FTP command
#   TCP 30000-31000 - Passive data

# Azure NSG inbound rules:
# Port 21: Allow
# Port range 30000-31000: Allow

# The Elastic IP / Public IP is used as pasv_address
# /etc/vsftpd.conf
pasv_address=203.0.113.10   # Elastic IP
```

## Verifying the PASV Response

```bash
# Manual FTP session to verify PASV response:
telnet 203.0.113.10 21
# 220 Welcome to FTP service
USER ftpuser
# 331 Please specify the password.
PASS secret
# 230 Login successful.
PASV
# 227 Entering Passive Mode (203,0,113,10,117,49)
#                            ^^^^^^^^^^^^^^
#                            Should be public IP

# Check vsftpd is advertising correct address:
# PASV format: (h1,h2,h3,h4,p1,p2)
# IP = h1.h2.h3.h4 = 203.0.113.10
# Port = p1*256 + p2 = 117*256 + 49 = 30001
```

## Conclusion

`pasv_address` is mandatory when vsftpd is behind NAT. Set it to your public IPv4 address so passive mode PASV responses advertise the reachable address. Use `pasv_addr_resolve=YES` with `pasv_address=your.hostname.com` for dynamic IP environments. Always forward both port 21 and the full passive port range through your NAT gateway.
