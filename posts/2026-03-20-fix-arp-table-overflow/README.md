# How to Fix ARP Table Overflow on Switches and Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ARP, ARP Table, Overflow, Switch, Router, Network

Description: Learn how to detect and fix ARP table overflow on switches and routers, where the hardware ARP table becomes full and new address resolutions fail, causing intermittent connectivity.

## What Is ARP Table Overflow?

Network switches and routers have hardware-limited ARP tables (CAM/TCAM tables). When the table is full:
- New ARP entries cannot be added
- New devices cannot communicate through the device
- Existing entries continue to work until they age out
- Logs show "ARP table full" or similar messages

## Step 1: Check ARP Table Usage

```bash
# Linux router/host - check ARP table

ip neigh show
ip neigh show | wc -l    # Count entries

# Check ARP table limits
sysctl net.ipv4.neigh.default.gc_thresh1    # Low watermark (GC starts)
sysctl net.ipv4.neigh.default.gc_thresh2    # High watermark
sysctl net.ipv4.neigh.default.gc_thresh3    # Maximum (hard limit)

# Example values:
# net.ipv4.neigh.default.gc_thresh1 = 128
# net.ipv4.neigh.default.gc_thresh2 = 512
# net.ipv4.neigh.default.gc_thresh3 = 1024  <- Maximum ARP entries
```

```text
! Cisco IOS
Router# show arp | count
Router# show arp summary
Router# show platform resources | include ARP
Router# show mac address-table count
```

## Step 2: Detect Overflow in Logs

```bash
# Linux kernel logs
dmesg | grep -i "neighbor table overflow"
# Look for: neighbour table overflow!

# Syslog
grep -i "arp" /var/log/syslog | grep -i "overflow\|full"

# Watch ARP failures in real-time
dmesg -w | grep neighbour
```

```text
! Cisco IOS - look for ARP overflow messages
Router# show log | include ARP
Router# show log | include TCAM
! Look for: %IP-4-ARPREQUESTLIMIT: ARP rate limit exceeded
```

## Step 3: Increase ARP Table Limits

```bash
# Linux - increase ARP table size for large networks
sudo tee -a /etc/sysctl.conf << 'EOF'

# ARP table limits
net.ipv4.neigh.default.gc_thresh1 = 1024   # Start GC at 1024 entries
net.ipv4.neigh.default.gc_thresh2 = 4096   # Aggressive GC above 4096
net.ipv4.neigh.default.gc_thresh3 = 8192   # Hard maximum

# ARP cache timeout (seconds)
net.ipv4.neigh.default.gc_stale_time = 60  # Remove stale entries after 60s
net.ipv4.neigh.default.base_reachable_time_ms = 30000  # 30s reachability
EOF

sudo sysctl -p
```

```text
! Cisco IOS - increase ARP timeout to reduce thrashing
Router(config)# arp timeout 1200    ! 20 minutes (default is 4 hours)

! On some platforms, adjust ARP table allocation
Router(config)# ip arp proxy disable   ! Disable proxy ARP if not needed
```

## Step 4: Reduce ARP Table Entries

```bash
# Identify what's filling the ARP table
ip neigh show | awk '{print $5}' | sort | uniq -c | sort -rn
# REACHABLE - active entries (needed)
# STALE - aged but not removed (check count)
# FAILED - failed resolutions (remove)
# NOARP - no ARP on interface

# Remove failed entries
ip neigh flush nud failed

# Remove stale entries
ip neigh flush nud stale

# Remove all entries (CAUTION: brief connectivity disruption)
ip neigh flush all
```

## Step 5: Address Root Cause - Oversized Subnets

```bash
# The most common cause: a /16 subnet with thousands of devices
# creates an ARP table with potentially 65534 entries

# Solution: segment large subnets into smaller /24s
# Before: 10.0.0.0/16 - one broadcast domain, up to 65534 hosts
# After: 10.0.1.0/24, 10.0.2.0/24, ... - max 254 hosts per ARP table

# Calculate how many subnets you need
python3 -c "
total_hosts = 5000
hosts_per_subnet = 254  # /24
subnets_needed = -(-total_hosts // hosts_per_subnet)  # ceiling division
print(f'Need {subnets_needed} x /24 subnets for {total_hosts} hosts')
"
```

## Step 6: Disable Proxy ARP Where Unnecessary

```bash
# Linux - disable proxy ARP (can cause extra entries)
sudo sysctl -w net.ipv4.conf.eth0.proxy_arp=0
sudo sysctl -w net.ipv4.conf.all.proxy_arp=0
```

```text
! Cisco IOS - disable proxy ARP per interface
Router(config)# interface GigabitEthernet0/0
Router(config-if)# no ip proxy-arp

! Verify
Router# show interfaces GigabitEthernet0/0 | include proxy
```

## Step 7: Monitor ARP Table Health

```bash
#!/bin/bash
# /usr/local/bin/arp-table-monitor.sh

MAX=$(sysctl -n net.ipv4.neigh.default.gc_thresh3)
CURRENT=$(ip neigh show | wc -l)
PCT=$(( CURRENT * 100 / MAX ))

echo "ARP table: $CURRENT/$MAX entries ($PCT% full)"

if [ $PCT -ge 80 ]; then
    logger -p daemon.warning "ARP table at $PCT% ($CURRENT/$MAX entries)"
fi
```

## Conclusion

ARP table overflow is detected via `dmesg | grep "neighbour table overflow"` and confirmed by comparing `ip neigh show | wc -l` against `gc_thresh3`. Fix immediately by flushing stale entries with `ip neigh flush nud stale` and increasing `gc_thresh3` in sysctl. The root cause is usually oversized subnets - segment large /16 networks into /24 subnets to keep individual ARP tables manageable. Disable proxy ARP where not needed to reduce unnecessary entries.
