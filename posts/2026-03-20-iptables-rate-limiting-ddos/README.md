# How to Set Up Rate Limiting with iptables to Prevent DDoS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: iptables, DDoS, Rate Limiting, IPv4, Linux, Security

Description: Use iptables rate limiting modules (limit, hashlimit, connlimit) to protect Linux servers against DDoS attacks and port scanning.

Rate limiting with iptables throttles incoming IPv4 connections to prevent resource exhaustion from DDoS attacks, brute force, and connection floods.

## Method 1: Global Rate Limit with --limit

The `limit` module limits packet rates globally:

```bash
# Allow only 5 ping requests per second (prevents ICMP flood)
sudo iptables -A INPUT -p icmp --icmp-type echo-request \
  -m limit --limit 5/second --limit-burst 10 -j ACCEPT
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j DROP

# Limit new SSH connections to 4 per minute globally
sudo iptables -A INPUT -p tcp --dport 22 \
  -m state --state NEW \
  -m limit --limit 4/minute --limit-burst 6 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -j DROP
```

## Method 2: Per-IP Rate Limit with --hashlimit

The `hashlimit` module limits rates per source IP:

```bash
# Allow each source IP maximum 10 connections per minute to SSH
sudo iptables -A INPUT -p tcp --dport 22 \
  -m state --state NEW \
  -m hashlimit \
  --hashlimit-name ssh-limit \
  --hashlimit-mode srcip \
  --hashlimit-upto 4/min \
  --hashlimit-burst 6 \
  --hashlimit-htable-expire 60000 \
  -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -j DROP

# Rate limit HTTP connections per IP (anti-scraping)
sudo iptables -A INPUT -p tcp --dport 80 \
  -m hashlimit \
  --hashlimit-name http-limit \
  --hashlimit-mode srcip \
  --hashlimit-upto 100/min \
  --hashlimit-burst 200 \
  -j ACCEPT
```

## Method 3: Connection Count Limit with --connlimit

Limit the number of concurrent connections from a single IP:

```bash
# Allow max 10 concurrent connections to SSH per source IP
sudo iptables -A INPUT -p tcp --dport 22 \
  -m connlimit --connlimit-above 10 -j REJECT

# Limit concurrent HTTP connections per IP to 20
sudo iptables -A INPUT -p tcp --dport 80 \
  -m connlimit --connlimit-above 20 -j REJECT
```

## SYN Flood Protection

```bash
# Limit SYN packets to prevent SYN flood
sudo iptables -A INPUT -p tcp --syn \
  -m limit --limit 50/second --limit-burst 100 -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP

# Or use SYN cookies (kernel-level):
sudo sysctl -w net.ipv4.tcp_syncookies=1
echo "net.ipv4.tcp_syncookies = 1" | sudo tee -a /etc/sysctl.conf
```

## Port Scan Detection

```bash
# Create a blocklist chain
sudo iptables -N PORTSCAN
sudo iptables -A PORTSCAN -m recent --set --name portscanners -j DROP

# Detect port scans (multiple rapid SYN attempts to different ports)
sudo iptables -A INPUT -p tcp --syn \
  -m recent --update --seconds 60 --hitcount 5 --name portscanners \
  -j PORTSCAN

sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

## Combined Anti-DDoS Script

```bash
#!/bin/bash
# anti-ddos.sh — Multi-layer rate limiting

# SYN flood protection
iptables -A INPUT -p tcp --syn -m limit --limit 50/s --limit-burst 100 -j ACCEPT

# Per-IP rate limiting for SSH
iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m hashlimit --hashlimit-name ssh --hashlimit-mode srcip \
  --hashlimit-upto 3/min --hashlimit-burst 5 -j ACCEPT

# Concurrent connection limits
iptables -A INPUT -p tcp --dport 80 -m connlimit --connlimit-above 50 -j REJECT
iptables -A INPUT -p tcp --dport 443 -m connlimit --connlimit-above 50 -j REJECT

# ICMP rate limiting
iptables -A INPUT -p icmp -m limit --limit 10/s -j ACCEPT
iptables -A INPUT -p icmp -j DROP

echo "Anti-DDoS rules applied"
```

## Monitoring Rate Limit Hits

```bash
# View hashlimit table
cat /proc/net/ipt_hashlimit/ssh-limit

# Check rule hit counts
sudo iptables -L INPUT -n -v | grep -E "ssh|http|icmp"
```

Rate limiting doesn't stop determined DDoS attacks entirely but prevents single-source floods and brute force from overwhelming the server.
