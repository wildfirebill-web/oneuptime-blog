# How to Fix "Network Cable Unplugged" Errors Caused by Duplex Mismatch

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Duplex Mismatch, Ethernet, ethtool, Network Troubleshooting, Linux, Cisco, Speed

Description: Learn how to diagnose and fix duplex mismatch issues that cause intermittent connectivity, high error rates, and "network cable unplugged" errors on Ethernet interfaces.

---

A duplex mismatch occurs when one side of an Ethernet link auto-negotiates to half-duplex while the other side is forced to full-duplex. Symptoms include late collisions, high FCS errors, and intermittent connectivity.

## Symptoms of Duplex Mismatch

```
- Interface shows UP but ping drops packets intermittently
- Very slow throughput (< 10% of link speed)
- High error/collision counters on one side
- "Late collisions" in interface statistics
- "Network cable unplugged" flapping on Linux
```

## Diagnosing with ethtool

```bash
# Check current speed and duplex
ethtool eth0

# Output:
# Settings for eth0:
#   Speed: 100Mb/s
#   Duplex: Half          ← mismatch if other side is Full
#   Auto-negotiation: off
#   Link detected: yes

# Check error counters
ethtool -S eth0 | grep -E "error|collision|miss|drop"
```

## Diagnosing with ip link

```bash
ip -s link show eth0
# RX errors and TX errors visible
# High TX errors with low RX errors = duplex mismatch
```

## Fixing on Linux: Force Speed and Duplex

```bash
# Force full-duplex at 100Mbps
ethtool -s eth0 speed 100 duplex full autoneg off

# Or re-enable auto-negotiation (preferred)
ethtool -s eth0 autoneg on

# Make persistent with udev rule
# /etc/udev/rules.d/10-ethtool.rules
ACTION=="add", SUBSYSTEM=="net", KERNEL=="eth0", \
  RUN+="/sbin/ethtool -s eth0 speed 100 duplex full autoneg off"
```

## Fixing on Cisco IOS

```
! Force speed and duplex on switch port
interface FastEthernet0/1
  speed 100
  duplex full
  no shutdown

! Verify
show interface FastEthernet0/1 | include duplex|speed|error
```

## Best Practice: Match Both Sides

```
Preferred: Both sides auto-negotiate
  Linux:  ethtool -s eth0 autoneg on
  Cisco:  speed auto / duplex auto

Forced (when one side doesn't support autoneg):
  Linux:  ethtool -s eth0 speed 1000 duplex full autoneg off
  Cisco:  speed 1000 / duplex full
```

## Checking Interface Errors Over Time

```bash
# Watch error counters
watch -n 2 "ip -s link show eth0 | grep -A4 RX"

# Check /proc for interface stats
cat /proc/net/dev | grep eth0
```

## Key Takeaways

- Duplex mismatch causes late collisions and high error rates; use `ethtool eth0` to verify speed and duplex.
- The safest fix is to enable auto-negotiation on both sides; avoid mixing forced and auto-negotiated settings.
- If forced settings are required, both sides must be set to the same speed and duplex explicitly.
- Use `ethtool -S eth0 | grep collision` to confirm late collisions as evidence of duplex mismatch.
