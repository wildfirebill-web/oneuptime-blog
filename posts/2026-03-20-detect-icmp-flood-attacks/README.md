# How to Detect ICMP Flood (Ping Flood) Attacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMP, DDoS, Security, Networking, iptables, Linux

Description: Detect and identify ICMP flood attacks using packet capture analysis, rate counters, and monitoring tools, then apply mitigations to protect your systems.

## Introduction

An ICMP flood (or ping flood) is a denial-of-service attack where an attacker sends a large volume of ICMP Echo Requests to a target, overwhelming its network interface or CPU. Modern systems can handle thousands of pings per second, so attacks usually come from botnets generating millions of packets per second. Detection is the first step to mitigation.

## Detecting an ICMP Flood

```bash
# Check incoming packet rate on an interface
watch -n 1 "ip -s link show eth0 | grep 'RX packets'"

# More detailed: monitor ICMP packet rate with tcpdump
tcpdump -i eth0 -n icmp 2>/dev/null | \
  awk '{count++} NR%100==0{print count " ICMP pkts/100 captures"}'

# Use iftop to see top talkers by IP
apt install iftop
iftop -i eth0 -f 'icmp' -n
```

## Identifying the Source

```bash
# Capture ICMP and show top source IPs (last 1000 packets)
tcpdump -i eth0 -n 'icmp[0]=8' -c 1000 | \
  awk '{print $3}' | sort | uniq -c | sort -rn | head -20

# In a flood, you'll see either:
# - One IP dominating (simple attack)
# - Thousands of IPs (distributed/amplification attack)

# Check if attack is amplification (requests from spoofed source IPs)
tcpdump -i eth0 -n 'icmp[0]=0' | head -20
# Replies going OUT = your server is amplifying (source IPs are spoofed)
```

## Monitoring with netstat and /proc

```bash
# Check ICMP statistics (includes receive rate)
cat /proc/net/snmp | grep "Icmp:"

# Watch ICMP counters in real time
watch -n 1 "cat /proc/net/snmp | grep Icmp:"
# Look for rapidly increasing InEchos counter

# Or use ss + nstat
nstat -a | grep Icmp
watch -n 1 "nstat -z | grep -i icmp"
```

## Setting Up Alerts

```bash
#!/bin/bash
# Monitor ICMP flood and alert if rate exceeds threshold
THRESHOLD=1000  # packets per second

while true; do
    BEFORE=$(cat /proc/net/snmp | awk '/^Icmp:/{getline; print $3}')
    sleep 1
    AFTER=$(cat /proc/net/snmp | awk '/^Icmp:/{getline; print $3}')
    RATE=$((AFTER - BEFORE))

    if [ "$RATE" -gt "$THRESHOLD" ]; then
        echo "$(date): ICMP FLOOD DETECTED - $RATE pps" | tee -a /var/log/icmp-flood.log
        # Add notification here (email, webhook, etc.)
    fi
done
```

## Immediate Mitigation

```bash
# Block all ICMP echo requests immediately
iptables -I INPUT -p icmp --icmp-type echo-request -j DROP

# More targeted: block from specific source
iptables -I INPUT -s 10.50.0.0/24 -p icmp -j DROP

# Rate-limit ICMP to 10 pps (prevents floods, allows monitoring)
iptables -I INPUT -p icmp --icmp-type echo-request \
  -m limit --limit 10/sec --limit-burst 20 -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
```

## Conclusion

ICMP floods are detectable through packet capture analysis and /proc counters. In a real flood, ICMP packet rates will jump by orders of magnitude above baseline. Immediate mitigation with iptables rate limiting stops the flood while preserving legitimate monitoring pings. For distributed attacks, coordinate with your ISP for upstream filtering or use a DDoS scrubbing service.
