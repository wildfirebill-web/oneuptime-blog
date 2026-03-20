# How to Set Up DNS over HTTPS (DoH) on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, DoH, HTTPS, Privacy, Security, Linux, Systemd-resolved

Description: Configure DNS over HTTPS on Linux using systemd-resolved, dnscrypt-proxy, or cloudflared to encrypt DNS queries and prevent eavesdropping.

## Introduction

DNS over HTTPS (DoH) sends DNS queries inside HTTPS traffic on port 443, preventing ISPs, network operators, and attackers from seeing or modifying your DNS queries. Unlike regular DNS (UDP port 53), DoH is indistinguishable from normal HTTPS traffic. This guide covers configuring DoH on Linux using multiple approaches.

## Method 1: systemd-resolved with DoT (DNS over TLS)

```bash
# systemd-resolved supports DNS over TLS (similar privacy, port 853):

# /etc/systemd/resolved.conf:
cat > /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com 8.8.8.8#dns.google
FallbackDNS=9.9.9.9#dns.quad9.net
DNSOverTLS=yes       # Force encrypted DNS
DNSSEC=yes           # Also validate DNSSEC
EOF

systemctl restart systemd-resolved

# Verify DoT is working:
resolvectl status | grep -E "DNS\|TLS\|Protocol"
# Should show: DNS over TLS: yes

# Test resolution still works:
dig google.com
```

## Method 2: dnscrypt-proxy for DoH

```bash
# Install dnscrypt-proxy:
apt-get install dnscrypt-proxy -y   # Ubuntu
# or: dnf install dnscrypt-proxy -y  # RHEL

# Configure dnscrypt-proxy:
cat > /etc/dnscrypt-proxy/dnscrypt-proxy.toml << 'EOF'
# Listen on localhost:5053 (so systemd-resolved still handles port 53)
listen_addresses = ['127.0.0.1:5053']

# Use DoH servers:
server_names = ['cloudflare', 'google', 'quad9-dnscrypt-ip4-filter-pri']

# Enable DoH specifically:
doh_servers = true
dnscrypt_servers = false  # Only DoH, no DNScrypt

# Logging:
[log]
level = 'notice'
file = '/var/log/dnscrypt-proxy.log'
EOF

# Start dnscrypt-proxy:
systemctl start dnscrypt-proxy
systemctl enable dnscrypt-proxy

# Point systemd-resolved to dnscrypt-proxy:
cat > /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=127.0.0.1:5053
DNSStubListener=yes
EOF

systemctl restart systemd-resolved
```

## Method 3: cloudflared (Argo Tunnel)

```bash
# Install cloudflared:
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
apt install ./cloudflared-linux-amd64.deb

# Run as DNS proxy:
cloudflared proxy-dns \
  --port 5053 \
  --upstream https://1.1.1.1/dns-query \
  --upstream https://1.0.0.1/dns-query

# Or create systemd service:
cat > /etc/systemd/system/cloudflared-dns.service << 'EOF'
[Unit]
Description=Cloudflare DNS over HTTPS proxy
After=network.target

[Service]
ExecStart=/usr/bin/cloudflared proxy-dns --port 5053 \
  --upstream https://1.1.1.1/dns-query \
  --upstream https://1.0.0.1/dns-query
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start cloudflared-dns
systemctl enable cloudflared-dns

# Configure systemd-resolved to use it:
# nameserver 127.0.0.1:5053 in resolved.conf
```

## Verify DoH is Working

```bash
# Test that DNS resolution still works:
dig @127.0.0.1 -p 5053 google.com

# Check if DNS traffic is encrypted (no UDP 53 queries should leave host):
# Method 1: Capture on external interface, should see NO UDP 53
tcpdump -i eth0 -n 'udp port 53' -c 5
# If you get captures: some DNS is still going out unencrypted

# Method 2: Verify with Cloudflare's test:
curl https://1.1.1.1/help
# Should show: "Using DNS over HTTPS (DoH): Yes" if using cloudflared

# Method 3: Check with Cloudflare DNS test:
curl https://www.cloudflare.com/cdn-cgi/trace
# Look for: DoH=1 in the output

# Verify no plaintext DNS leaks:
sudo tcpdump -i eth0 -n 'port 53' -c 30 &
# Browse for 10 seconds:
for i in $(seq 1 10); do dig google.com > /dev/null; sleep 1; done
# If no packets captured on port 53: DoH is working
```

## Public DoH Servers

```yaml
Provider      | URL                              | Notes
--------------|----------------------------------|------------------
Cloudflare    | https://1.1.1.1/dns-query        | No logging option: 1.1.1.2
Google        | https://dns.google/dns-query     | Full Google logging
Quad9         | https://dns.quad9.net/dns-query  | Malware filtering
NextDNS       | https://dns.nextdns.io/          | Custom filtering
Mullvad       | https://adblock.dns.mullvad.net/ | Privacy-focused
```

## Conclusion

DNS over HTTPS encrypts your DNS queries inside standard HTTPS, preventing observation by ISPs and network monitors. Use `systemd-resolved` with `DNSOverTLS=yes` for the simplest setup (though this is technically DoT, not DoH). For true DoH, use `dnscrypt-proxy` or `cloudflared` as a local proxy, then point your resolver to it. Verify there are no DNS leaks by capturing on the external interface and confirming no UDP port 53 traffic leaves the host.
