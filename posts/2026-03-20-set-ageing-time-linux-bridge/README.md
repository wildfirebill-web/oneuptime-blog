# How to Set Ageing Time on a Linux Bridge

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Bridge, Ageing Time, FDB, Linux, Brctl, MAC Address Table, Networking

Description: Learn how to configure the MAC address ageing time on a Linux bridge to control how long learned MAC-to-port mappings are retained in the forwarding database.

---

Bridge ageing time determines how long the bridge remembers which port a MAC address was last seen on. After the timeout, the entry is removed and the next frame to that MAC is flooded.

## What is Ageing Time?

```text
MAC table entry: aa:bb:cc:dd:ee:ff → port eth0 (learned 120 seconds ago)
                                          ↑
                 If no traffic for <ageing_time> seconds: entry deleted
                 Next frame to this MAC: flooded to all ports
```

- Default: 300 seconds (5 minutes)
- Lower value: More flooding, fresher table (good for high-change environments)
- Higher value: Less flooding, potentially stale entries (good for stable environments)

## Viewing Current Ageing Time

```bash
# Show current ageing time

brctl showstp br0 | grep "ageing time"
# ageing time               300.00

# Via sysfs
cat /sys/class/net/br0/bridge/ageing_time
# Value in jiffies (100 = 1 second on most systems)
# 30000 jiffies / 100 = 300 seconds
```

## Setting Ageing Time with brctl

```bash
# Set ageing time to 60 seconds
brctl setageing br0 60

# Disable ageing (never expire - use with caution)
brctl setageing br0 0

# Verify
brctl showstp br0 | grep "ageing time"
```

## Setting Ageing Time with ip link

```bash
# Set ageing time (in seconds)
ip link set br0 type bridge ageing_time 6000   # 60 seconds (value in centiseconds)
# Note: ip link uses centiseconds (1/100 second)
# 60 seconds = 6000 centiseconds

# Verify
ip -d link show br0 | grep ageing
```

## Persistent Configuration (systemd-networkd)

```ini
# /etc/systemd/network/br0.netdev
[NetDev]
Name=br0
Kind=bridge

[Bridge]
AgeingTimeSec=60
```

## Persistent with /etc/network/interfaces (Debian)

```bash
auto br0
iface br0 inet static
  address 192.168.1.10
  netmask 255.255.255.0
  bridge_ports eth0
  bridge_stp on
  bridge_fd 0
  bridge_ageing 60    # 60 seconds
```

## Viewing and Clearing the FDB

```bash
# View all FDB entries with age
bridge fdb show br br0

# Check FDB entry count
bridge fdb show br br0 | wc -l

# Flush all dynamic entries (forces relearning)
bridge fdb flush dev br0
```

## When to Adjust Ageing Time

| Scenario | Recommended Ageing |
|----------|-------------------|
| Default (stable LAN) | 300 seconds |
| Live VM migration | 5-30 seconds (quick MAC move detection) |
| High-churn WiFi | 60-120 seconds |
| Static environments | 600+ seconds (less flooding) |

## Key Takeaways

- Ageing time controls how long the bridge retains MAC-to-port mappings in the forwarding database.
- Use `brctl setageing br0 <seconds>` or `ip link set br0 type bridge ageing_time <centiseconds>`.
- Reduce ageing time during live VM migration to accelerate MAC address table updates on the bridge.
- `bridge fdb flush dev br0` forcefully clears the MAC table, useful after topology changes.
