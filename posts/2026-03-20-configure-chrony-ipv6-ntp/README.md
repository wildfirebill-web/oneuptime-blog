# How to Configure chrony for IPv6 NTP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Chrony, NTP, IPv6, Time Synchronization, Linux, Chronyd

Description: Configure chronyd (chrony) to synchronize time over IPv6, serve NTP to IPv6 clients, and use IPv6-accessible NTP pool servers on Linux systems.

---

chrony is the modern NTP implementation used by default on RHEL, CentOS, Fedora, and newer Ubuntu systems. It has excellent IPv6 support and is more accurate than the traditional ntpd implementation.

## Installing chrony

```bash
# RHEL/CentOS/AlmaLinux/Fedora

sudo dnf install chrony -y

# Ubuntu/Debian
sudo apt install chrony -y

# Check version and verify IPv6 support
chronyc --version
```

## Basic chrony IPv6 Configuration

```bash
# /etc/chrony.conf

# Use NTP pool servers (these support IPv6)
pool pool.ntp.org iburst maxsources 4

# Or use specific IPv6 NTP servers
server 2001:db8:ntp::1 iburst
server time.google.com iburst    # Has AAAA record
server time.cloudflare.com iburst # Has AAAA record

# Prefer IPv6 for time sources
# chrony automatically uses IPv6 when available via DNS AAAA records
```

## Configuring chrony to Serve NTP to IPv6 Clients

```bash
# /etc/chrony.conf

# Allow IPv6 clients from your subnet
allow 2001:db8::/32

# Allow all IPv6 (not recommended for public servers)
# allow ::/0

# Allow specific IPv6 host
allow 2001:db8::100

# Combine with IPv4 access
allow 10.0.0.0/8
allow 192.168.0.0/16

# Bind to a specific IPv6 address for serving NTP
bindaddress 2001:db8::1

# Or listen on all interfaces (default)
# bindaddress ::
```

## Forcing IPv6 or IPv6-Only Operation

```bash
# /etc/chrony.conf

# Force chrony to only use IPv6 for NTP sources
# Use -6 flag in the server/pool directive
server ipv6.time.example.com iburst version 4

# For the chronyc command, force IPv6
chronyc -6 sources

# To restrict chronyd to IPv6 only at daemon level,
# add to /etc/sysconfig/chronyd or systemd override:
# OPTIONS="-6"
```

Create a systemd override to force IPv6:

```bash
# Create override directory
sudo mkdir -p /etc/systemd/system/chronyd.service.d/

# Create override file
cat > /etc/systemd/system/chronyd.service.d/ipv6.conf << 'EOF'
[Service]
# Force chrony to use only IPv6 for NTP
ExecStart=
ExecStart=/usr/sbin/chronyd $OPTIONS -6
EOF

sudo systemctl daemon-reload
sudo systemctl restart chronyd
```

## Verifying chrony IPv6 Operation

```bash
# Check chrony sources (should show IPv6 addresses)
chronyc sources -v

# Check tracking information
chronyc tracking

# Check if chronyd is listening on IPv6
ss -ulnp | grep :123
# Look for [::]:123 in the output

# Query a specific IPv6 NTP server
chronyc -h 2001:db8::1 sources

# Check NTP activity on network interface
tcpdump -i eth0 -n ip6 and udp port 123
```

## chrony Access Control and Security

```bash
# /etc/chrony.conf - Security hardened configuration

# Upstream sources
pool pool.ntp.org iburst maxsources 4

# Only allow NTP queries from trusted IPv6 subnets
allow 2001:db8:internal::/48

# Deny queries from all other IPv6 addresses (explicit)
deny ::/0

# Require NTP authentication from clients
# (see NTP authentication guide for key setup)
# keyfile /etc/chrony.keys

# Rate limit clients to prevent amplification attacks
ratelimit interval 3 burst 8

# Log client access
logdir /var/log/chrony
log tracking measurements statistics
```

## Using chronyc for IPv6 Diagnostics

```bash
# List current NTP sources with IPv6 details
chronyc -n sources

# Check server clients (who is using this server)
chronyc clients

# Manually trigger time step
sudo chronyc makestep

# Add a new NTP source at runtime
sudo chronyc add server 2001:db8::ntp1 iburst

# Check activity statistics
chronyc activity

# Verify time offset
chronyc -n tracking | grep "System time"
```

## Troubleshooting chrony IPv6 Issues

```bash
# Check chrony log for errors
sudo journalctl -u chronyd -f

# Look for IPv6-specific issues
sudo journalctl -u chronyd | grep -i "ipv6\|inet6\|resolve\|unreachable"

# Test DNS resolution of NTP pool to IPv6
dig AAAA pool.ntp.org +short

# Manually test NTP response from an IPv6 server
ntpdate -q 2001:db8::1

# Check kernel NTP status
timedatectl status
timedatectl show-timesync --all
```

chrony is the recommended NTP implementation for modern Linux systems, offering precise time synchronization over IPv6 with better accuracy, lower latency detection, and more robust handling of variable network conditions than legacy ntpd.
