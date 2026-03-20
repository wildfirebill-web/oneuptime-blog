# How to Configure IPv4 DNS Settings for WiFi Clients on a Router

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, WiFi, DHCP, Router, IPv4, Configuration

Description: Learn how to configure DNS server settings that are distributed to WiFi clients via DHCP, including setting primary/secondary DNS and deploying split-DNS for internal resources.

## How DNS Is Distributed to WiFi Clients

The router distributes DNS server IP addresses to WiFi clients as part of the DHCP process. The client uses these DNS servers for all name resolution until the DHCP lease expires or the DNS is manually overridden.

## Step 1: Configure DNS on Common Router Types

**Consumer Router (OpenWrt):**
```bash
# /etc/config/dhcp

config dnsmasq
    option domainneeded '1'
    option boguspriv '1'
    option filterwin2k '0'
    option localise_queries '1'
    option rebind_protection '1'
    option local '/lan/'
    option domain 'lan'
    option expandhosts '1'

# DHCP pool with DNS options
config dhcp 'lan'
    option interface 'lan'
    option start '100'
    option limit '150'
    option leasetime '12h'
    # Push DNS servers to clients
    list dhcp_option '6,8.8.8.8,8.8.4.4'
```

**ISC DHCPD (Linux):**
```bash
# /etc/dhcp/dhcpd.conf
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;

    # DNS options - specify one or more DNS servers
    option domain-name-servers 8.8.8.8, 8.8.4.4;

    # Optional: push domain search list
    option domain-name "example.local";
    option domain-search "example.local", "corp.example.com";
}
```

## Step 2: Set Up a Local DNS Server (dnsmasq)

Using the router as a local DNS cache/resolver:

```bash
# /etc/dnsmasq.conf

# Use router as DNS (clients get router's IP as DNS server)
listen-address=127.0.0.1,192.168.1.1
bind-interfaces

# Forward to upstream DNS servers
server=8.8.8.8
server=8.8.4.4

# Cache size (number of cached DNS entries)
cache-size=10000

# Local DNS overrides for internal hosts
address=/fileserver.local/192.168.1.10
address=/printer.local/192.168.1.20
address=/nas.local/192.168.1.30

# Don't forward .local queries to upstream
local=/local/
```

## Step 3: Configure Split-DNS for Corporate Environments

Split-DNS serves internal DNS for corporate domains and forwards everything else to public DNS:

```bash
# /etc/dnsmasq.conf

# Forward internal domain queries to internal DNS server
server=/corp.example.com/10.0.0.53
server=/example.internal/10.0.0.53

# Forward everything else to public DNS
server=8.8.8.8
server=1.1.1.1

# Clients get router IP as DNS server
# Router handles the split routing transparently
```

## Step 4: Use DNS over HTTPS (DoH) for WiFi Clients

For encrypted DNS resolution:

```bash
# Install cloudflared (Cloudflare DNS proxy)
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64
chmod +x cloudflared-linux-amd64
sudo mv cloudflared-linux-amd64 /usr/local/bin/cloudflared

# Configure as a DNS proxy
cloudflared proxy-dns --port 5053 &

# Point dnsmasq to the local DoH proxy
echo "server=127.0.0.1#5053" >> /etc/dnsmasq.conf
systemctl restart dnsmasq

# DHCP clients still get 192.168.1.1 as their DNS server
# The router then uses DoH for upstream resolution
```

## Step 5: Configure DNS on Individual WiFi Clients

Override router-provided DNS on individual clients:

**Linux:**
```bash
# Override via /etc/resolv.conf
echo "nameserver 1.1.1.1" > /etc/resolv.conf

# Or via NetworkManager
nmcli connection modify "WiFi-Network" ipv4.dns "1.1.1.1 8.8.8.8"
nmcli connection up "WiFi-Network"
```

**macOS:**
```bash
sudo networksetup -setdnsservers "Wi-Fi" 1.1.1.1 8.8.8.8
```

## Step 6: Test DNS Configuration

```bash
# Test DNS resolution time
time nslookup google.com 192.168.1.1    # Using router DNS
time nslookup google.com 8.8.8.8        # Using Google DNS directly

# Test internal DNS resolution
nslookup fileserver.local 192.168.1.1

# Verify clients are getting correct DNS
# On client:
ipconfig /all | findstr "DNS Servers"    # Windows
nmcli dev show wlan0 | grep DNS          # Linux
scutil --dns | grep nameserver           # macOS
```

## Conclusion

WiFi clients receive DNS server IPs from DHCP. Configure them in `dhcpd.conf` with `option domain-name-servers` or in `dnsmasq.conf` with `dhcp-option=6,...`. Run dnsmasq on the router as a local caching resolver for better performance and to support split-DNS for internal domain resolution. Test DNS with `nslookup` pointing at the router IP to confirm clients receive correct DNS service.
