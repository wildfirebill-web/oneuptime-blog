# How to Troubleshoot Network Bonding Failover Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Bonding, Failover, Troubleshooting, MII, Networking, High Availability

Description: Diagnose and fix network bond failover problems including slow failover, failed failover, and incorrect slave selection on Linux.

## Introduction

Bonding failover issues manifest as connectivity loss during link failures, slow failover times, or bonds failing to recover when a link comes back up. Systematic troubleshooting using `/proc/net/bonding`, kernel logs, and MII monitoring parameters resolves most issues.

## Step 1: Verify the Bond is in the Correct Mode

```bash
cat /proc/net/bonding/bond0 | grep "Bonding Mode"

# Failover-capable modes:
# fault-tolerance (active-backup) — Mode 1
# IEEE 802.3ad (LACP) — Mode 4
# adaptive load balancing — Mode 5/6
```

## Step 2: Check MII Monitoring is Enabled

MII monitoring is what detects link failures. Without it, the bond won't detect failures:

```bash
cat /proc/net/bonding/bond0 | grep "MII Polling"
# MII Polling Interval (ms): 100

# If interval is 0, monitoring is disabled!
# Enable it:
echo 100 > /sys/class/net/bond0/bonding/miimon
```

## Step 3: Check updelay and downdelay

Incorrect `updelay` and `downdelay` settings cause slow failover:

```bash
cat /proc/net/bonding/bond0 | grep -E "Up Delay|Down Delay"
# Up Delay (ms): 0
# Down Delay (ms): 0

# If downdelay is very high (e.g., 5000ms), failover will be slow
# Reset to 0 or 200ms
echo 200 > /sys/class/net/bond0/bonding/downdelay
echo 200 > /sys/class/net/bond0/bonding/updelay
```

## Step 4: Check Slave Link States

```bash
# Check each slave's MII state
grep -A 5 "Slave Interface" /proc/net/bonding/bond0

# Check link failure counts
grep "Link Failure Count" /proc/net/bonding/bond0
# High count indicates flapping
```

## Step 5: Test Failover Manually

```bash
# Before failover
echo "Active slave: $(cat /sys/class/net/bond0/bonding/active_slave)"

# Simulate failure: bring down the active slave
ip link set eth0 down

# Check failover happened (within miimon interval)
sleep 0.5
echo "Active slave after failure: $(cat /sys/class/net/bond0/bonding/active_slave)"
# Should now show eth1

# Verify connectivity during failover
ping -I bond0 8.8.8.8 -c 10 &

# Bring back eth0 and verify failback
ip link set eth0 up
sleep 1
echo "Active slave after recovery: $(cat /sys/class/net/bond0/bonding/active_slave)"
```

## Step 6: Check Kernel Logs for Failover Events

```bash
# Check dmesg for bond events
dmesg | grep -i "bond\|enslav\|freed"

# Monitor in real time
journalctl -kf | grep bond0
```

## Common Issues and Fixes

| Issue | Cause | Fix |
|---|---|---|
| No failover | miimon = 0 | `echo 100 > /sys/class/net/bond0/bonding/miimon` |
| Slow failover (>5s) | downdelay too high | `echo 200 > .../bonding/downdelay` |
| No failback to primary | primary_reselect = failure | `echo always > .../bonding/primary_reselect` |
| Bond stays with backup | Primary not set | `echo eth0 > .../bonding/primary` |
| Flapping interfaces | updelay too low | `echo 500 > .../bonding/updelay` |

## Set primary_reselect Policy

```bash
# Always return to primary when it comes back up
echo always > /sys/class/net/bond0/bonding/primary_reselect

# Return to primary only if it is better (higher speed)
echo better > /sys/class/net/bond0/bonding/primary_reselect

# Only return if the current active slave fails
echo failure > /sys/class/net/bond0/bonding/primary_reselect
```

## Conclusion

Bonding failover issues are almost always caused by miimon not being set, incorrect delay values, or primary_reselect configuration. Enable miimon at 100ms for fast detection, set updelay and downdelay to 200ms to avoid flapping, and configure primary_reselect to `always` for predictable failback behavior. Monitor failover events in the kernel log with `journalctl -kf | grep bond`.
