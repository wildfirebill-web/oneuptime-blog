# How to Flush DNS Cache on Linux, macOS, and Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, Cache, Linux, macOS, Windows, Troubleshooting

Description: Clear DNS cache on Linux (systemd-resolved, nscd, dnsmasq), macOS, and Windows to force fresh DNS lookups after record changes.

## Introduction

DNS caches store recently resolved names to improve performance. When DNS records change - during migrations, failovers, or IP changes - old cached entries prevent clients from picking up the new values until the TTL expires. Flushing the DNS cache forces an immediate re-query, allowing clients to resolve to new IP addresses without waiting for TTL expiration.

## Linux - systemd-resolved

```bash
# Most modern Linux distributions use systemd-resolved:

# Check if it's running:
systemctl status systemd-resolved

# Flush all DNS caches:
resolvectl flush-caches

# Verify cache was flushed (counter should reset to 0):
resolvectl statistics

# Flush for a specific interface:
resolvectl flush-caches eth0

# Check what's cached:
resolvectl query google.com  # This will force a lookup if not cached
```

## Linux - nscd (Name Service Cache Daemon)

```bash
# Check if nscd is running:
systemctl status nscd

# Flush all nscd caches:
nscd -i hosts     # Flush only host/DNS cache
# Or restart:
systemctl restart nscd

# Flush all cache types:
nscd -i passwd
nscd -i group
nscd -i hosts

# Check nscd statistics:
nscd -g
```

## Linux - dnsmasq

```bash
# Check if dnsmasq is running:
systemctl status dnsmasq

# Flush dnsmasq cache (sends SIGUSR1 to reload):
pkill -SIGUSR1 dnsmasq
# Or:
systemctl restart dnsmasq

# Note: SIGUSR1 causes dnsmasq to dump its cache stats to syslog,
# but does NOT flush the cache. To actually flush:
systemctl restart dnsmasq  # Only way to flush dnsmasq cache
```

## Linux - General (All Resolvers)

```bash
# Nuclear option: restart all DNS-related services:
systemctl restart systemd-resolved
systemctl restart nscd 2>/dev/null
systemctl restart dnsmasq 2>/dev/null

# Verify the new record resolves correctly after flush:
dig google.com
# Check: TTL decreasing confirms it was cached; high TTL = fresh lookup

# Check if a domain resolves to new IP after flush:
OLD_IP="93.184.216.34"
NEW_IP="1.2.3.4"
RESOLVED=$(dig +short example.com | head -1)
echo "Resolved to: $RESOLVED"
```

## macOS

```bash
# macOS cache flush command (varies by OS version):

# macOS 10.15 Catalina and later:
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder

# macOS 10.14 Mojave:
sudo killall -HUP mDNSResponder

# macOS 10.12-10.13:
sudo killall -HUP mDNSResponder

# Also reset NSCD:
sudo dscacheutil -flushcache

# Verify with:
dscacheutil -q host -a name google.com
# Should force a fresh lookup
```

## Windows

```powershell
# Command Prompt or PowerShell (Run as Administrator):
ipconfig /flushdns

# Verify the cache was flushed:
ipconfig /displaydns

# Restart DNS Client service for thorough flush:
net stop dnscache
net start dnscache

# PowerShell equivalent:
Clear-DnsClientCache

# Check what's currently cached:
Get-DnsClientCache | Where-Object {$_.Entry -match "example.com"}
```

## Verify the Flush Worked

```bash
# After flushing, confirm fresh lookup occurs:
# Method 1: Check TTL is at maximum (newly fetched record has full TTL):
dig example.com
# TTL should be close to the record's configured TTL (e.g., 300)
# A low TTL means it was cached and is expiring

# Method 2: Compare before/after flush:
IP_BEFORE=$(dig +short example.com)
resolvectl flush-caches
IP_AFTER=$(dig +short example.com)
echo "Before: $IP_BEFORE"
echo "After:  $IP_AFTER"
# If IPs differ: new record was picked up after flush
```

## Conclusion

DNS cache flushing procedures differ by OS and resolver: use `resolvectl flush-caches` on systemd-resolved Linux, `sudo killall -HUP mDNSResponder` on macOS, and `ipconfig /flushdns` on Windows. After flushing, verify the correct IP is returned. For production DNS record changes, set the TTL low (e.g., 60 seconds) before the change to minimize cache propagation time - this is more reliable than relying on clients to flush their caches.
