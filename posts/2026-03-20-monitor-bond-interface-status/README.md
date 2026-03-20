# How to Monitor Bond Interface Status on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Bonding, Monitoring, /proc/net/bonding, Networking, System Administration

Description: Monitor the health and status of Linux bond interfaces using /proc/net/bonding, ip commands, and custom monitoring scripts to detect failovers and slave link issues.

## Introduction

Monitoring bond interfaces ensures you know when a slave link fails, a failover occurs, or when LACP negotiation has issues. Linux exposes detailed bond status through `/proc/net/bonding/` and sysfs, which can be read with simple commands or automated monitoring scripts.

## Read the Bond Status File

The primary source of bond status information is `/proc/net/bonding/<bond_name>`:

```bash
cat /proc/net/bonding/bond0
```

Example output for active-backup bond:
```
Ethernet Channel Bonding Driver: v3.7.1

Bonding Mode: fault-tolerance (active-backup)
Primary Slave: eth0
Currently Active Slave: eth0
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: eth0
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 0

Slave Interface: eth1
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 0
```

## Key Status Fields

```bash
# Check the currently active slave
cat /sys/class/net/bond0/bonding/active_slave

# Check all slave interfaces
cat /sys/class/net/bond0/bonding/slaves

# Check bonding mode
cat /sys/class/net/bond0/bonding/mode

# Check MII polling interval
cat /sys/class/net/bond0/bonding/miimon
```

## Monitor Link Failure Count

```bash
# Extract link failure counts for all slaves
grep "Link Failure Count" /proc/net/bonding/bond0

# Watch for changes
watch -n 5 "grep -A 5 'Slave Interface' /proc/net/bonding/bond0"
```

## Automated Bond Health Monitor Script

```bash
#!/bin/bash
# bond-monitor.sh: Check bond health and alert on failures

BOND="bond0"
BOND_FILE="/proc/net/bonding/$BOND"

check_bond_health() {
    if [ ! -f "$BOND_FILE" ]; then
        echo "ERROR: Bond $BOND not found"
        exit 1
    fi

    # Check overall bond MII status
    BOND_STATUS=$(grep "^MII Status" "$BOND_FILE" | head -1 | awk '{print $3}')
    echo "Bond status: $BOND_STATUS"

    # Check each slave
    while IFS= read -r line; do
        if [[ "$line" =~ "Slave Interface:" ]]; then
            SLAVE=$(echo "$line" | awk '{print $3}')
        fi
        if [[ "$line" =~ "MII Status:" && -n "$SLAVE" ]]; then
            STATUS=$(echo "$line" | awk '{print $3}')
            FAILURES=$(grep -A 5 "Slave Interface: $SLAVE" "$BOND_FILE" | \
                       grep "Link Failure Count" | awk '{print $4}')
            echo "Slave $SLAVE: $STATUS (failures: $FAILURES)"

            if [ "$STATUS" != "up" ]; then
                echo "ALERT: Slave $SLAVE is DOWN!"
            fi
            SLAVE=""
        fi
    done < "$BOND_FILE"
}

check_bond_health
```

## Monitor with ip Commands

```bash
# Check interface state
ip link show bond0
ip link show eth0
ip link show eth1

# Detailed interface statistics
ip -s link show bond0
```

## Watch Bond in Real Time

```bash
# Watch bond status updating every 2 seconds
watch -n 2 "cat /proc/net/bonding/bond0"
```

## Log Failover Events via Syslog

```bash
# Bond failover events are logged to the kernel ring buffer
dmesg | grep -i bond

# Watch for real-time failover events
journalctl -kf | grep -i bond
```

## Conclusion

Bond status monitoring relies primarily on `/proc/net/bonding/<bond>` and sysfs. Watch for `MII Status: down` on slave interfaces and `Currently Active Slave` changes to detect failovers. Automate monitoring with scripts that check failure counts and alert when slaves go down. Kernel failover events are logged to syslog for post-incident analysis.
