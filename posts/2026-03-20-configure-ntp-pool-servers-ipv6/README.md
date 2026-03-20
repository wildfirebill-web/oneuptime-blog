# How to Configure NTP Pool Servers over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NTP, NTP Pool, IPv6, Time Synchronization, chrony, ntpd

Description: Configure your systems to use the NTP Pool Project servers over IPv6, including selecting IPv6-specific pool zones and verifying IPv6 connectivity to pool servers.

---

The NTP Pool Project (pool.ntp.org) is a global network of thousands of volunteer NTP servers. Many pool servers support IPv6, and the pool has dedicated IPv6 zones. Using IPv6 NTP pool servers reduces latency and enables time synchronization on IPv6-only networks.

## NTP Pool IPv6 Zones

The NTP Pool Project provides IPv6-specific zones:

```
# Global IPv6 pool
2.pool.ntp.org           # AAAA records (IPv6) plus A records
ipv6.pool.ntp.org        # IPv6-only pool

# Regional IPv6 pools
ipv6.asia.pool.ntp.org
ipv6.europe.pool.ntp.org
ipv6.north-america.pool.ntp.org
ipv6.oceania.pool.ntp.org
ipv6.south-america.pool.ntp.org

# Country-specific (if they have IPv6 nodes)
ipv6.us.pool.ntp.org
ipv6.de.pool.ntp.org
ipv6.jp.pool.ntp.org
```

## Checking IPv6 Availability of Pool Servers

```bash
# Verify the pool resolves to IPv6 addresses
dig AAAA pool.ntp.org +short
dig AAAA ipv6.pool.ntp.org +short

# Check specific regional pool
dig AAAA ipv6.europe.pool.ntp.org +short

# Verify connectivity to the IPv6 pool
ping6 -c 3 ipv6.pool.ntp.org
```

## Configuring chrony with IPv6 Pool Servers

```bash
# /etc/chrony.conf

# Use the IPv6-specific pool zone
pool ipv6.pool.ntp.org iburst maxsources 4

# Fallback to general pool (which also has IPv6)
pool pool.ntp.org iburst maxsources 2

# Or use regional IPv6 pool for better latency
pool ipv6.north-america.pool.ntp.org iburst maxsources 4

# Allow your subnet to use this server as NTP
allow 2001:db8::/32
allow 192.168.0.0/16

driftfile /var/lib/chrony/drift
logdir /var/log/chrony
```

```bash
# Apply configuration
sudo systemctl restart chronyd

# Verify sources include IPv6 addresses
chronyc sources -v
# IPv6 addresses will show in brackets: [2001:db8::x]
```

## Configuring ntpd with IPv6 Pool Servers

```bash
# /etc/ntp.conf

# Use IPv6 NTP pool
pool ipv6.pool.ntp.org iburst

# Or mix with standard pool (prefer IPv6 via DNS AAAA)
pool 0.pool.ntp.org iburst
pool 1.pool.ntp.org iburst
pool 2.pool.ntp.org iburst
pool 3.pool.ntp.org iburst

# Force IPv6 for specific servers using the -6 flag (ntpd option)
# ntpd resolves pool entries and uses AAAA if available

# Access restrictions
restrict default kod limited nomodify notrap nopeer noquery
restrict 127.0.0.1
restrict ::1
restrict 2001:db8::/32 nomodify notrap nopeer

driftfile /var/lib/ntp/drift
```

## Configuring systemd-timesyncd with IPv6 Pool

```bash
# /etc/systemd/timesyncd.conf

[Time]
# Use IPv6 NTP pool
NTP=ipv6.pool.ntp.org pool.ntp.org
FallbackNTP=time.google.com time.cloudflare.com
```

## Joining the NTP Pool as an IPv6 Server

To contribute an IPv6 NTP server to the pool:

```bash
# Step 1: Configure chrony as an accurate NTP server
# /etc/chrony.conf
pool ipv6.pool.ntp.org iburst maxsources 4
allow ::/0         # Allow pool monitoring connections
allow 0.0.0.0/0

# Step 2: Ensure your server has a static IPv6 address with proper DNS
dig AAAA your-ntp-server.example.com +short

# Step 3: Configure reverse DNS for your IPv6 address
# Contact your IP provider for PTR record setup

# Step 4: Register at https://www.ntppool.org/manage
# Add your IPv6 address when registering

# Verify your server is accessible before registering
ntpdate -q 2001:db8::your-server
```

## Monitoring Pool Server Performance

```bash
# Monitor which pool servers are being used
chronyc sources

# Check synchronization quality
chronyc sourcestats

# Verify time offset is acceptable
chronyc tracking | grep "System time"

# Watch NTP traffic to pool servers (IPv6)
sudo tcpdump -i eth0 -n ip6 and udp port 123
```

## Handling Dual-Stack NTP Pool Resolution

```bash
# Verify your system prefers IPv6 for NTP pool DNS
# Check /etc/gai.conf for address preference

cat /etc/gai.conf | grep "^precedence"
# Default usually prefers IPv6 when available

# Force IPv6 preference in gai.conf
echo "precedence ::ffff:0:0/96 5" | sudo tee -a /etc/gai.conf
# This setting prioritizes IPv4 for IPv4-mapped addresses

# To prefer native IPv6, ensure this line is present:
# precedence  ::1/128       50
# precedence  ::/0          40
```

Using the IPv6-specific NTP pool zones ensures your time synchronization traffic stays on IPv6, reduces cross-protocol overhead, and supports the global pool ecosystem by using IPv6-capable pool members.
