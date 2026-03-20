# How to Monitor IGMP Join and Leave Events in Real Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Multicast, IGMP, Monitoring, tcpdump, Linux

Description: Monitor IGMP Join and Leave events in real time using tcpdump, tshark, and kernel proc files to track multicast group membership changes on your network segment.

## Introduction

Watching IGMP Join and Leave events in real time lets you verify that receivers are correctly joining groups, diagnose spurious leave storms, and confirm that IGMP membership tables stay accurate. This guide covers several tools for live monitoring.

## Monitoring with tcpdump

```bash
# Capture all IGMP events on eth0 with timestamps and full decode

sudo tcpdump -i eth0 -n -v -l "ip proto 2" 2>/dev/null
```

The `-l` flag flushes output line-by-line, making it suitable for piping to other tools.

Example output:

```text
14:02:11.001234 IP 192.168.1.50 > 224.0.0.22: igmp v3 report, 1 group record(s)
  group address 239.1.2.3, mode IS_EXCLUDE, 0 sources
14:02:45.678901 IP 192.168.1.50 > 224.0.0.2: igmp leave 239.1.2.3
```

## Filtering Joins and Leaves Separately

```bash
# Watch only IGMPv2 Leave messages
sudo tcpdump -i eth0 -n -v "igmp" 2>/dev/null | grep -i "leave"

# Watch only Membership Reports (joins)
sudo tcpdump -i eth0 -n -v "igmp" 2>/dev/null | grep -i "report"
```

## Using tshark for Structured Output

`tshark` (Wireshark CLI) can print structured IGMP fields:

```bash
# Install tshark if needed
sudo apt install -y tshark

# Print timestamp, source, destination, and IGMP type
sudo tshark -i eth0 -n \
  -Y "igmp" \
  -T fields \
  -e frame.time \
  -e ip.src \
  -e ip.dst \
  -e igmp.type \
  -e igmp.maddr \
  2>/dev/null
```

IGMP type codes:

| Hex | Meaning |
|---|---|
| 0x11 | Membership Query |
| 0x16 | IGMPv2 Membership Report (Join) |
| 0x17 | IGMPv2 Leave Group |
| 0x22 | IGMPv3 Membership Report |

## Polling /proc/net/igmp for Changes

Write a shell script to detect membership changes by diffing `/proc/net/igmp`:

```bash
#!/bin/bash
# Monitor IGMP membership changes by polling /proc/net/igmp every 2 seconds

PREV=""
while true; do
    CURR=$(cat /proc/net/igmp)
    if [ "$CURR" != "$PREV" ]; then
        echo "=== IGMP membership changed at $(date) ==="
        diff <(echo "$PREV") <(echo "$CURR") | grep "^[<>]"
        PREV="$CURR"
    fi
    sleep 2
done
```

## Logging IGMP Events to a File

```bash
# Log all IGMP events to a timestamped file (rotate with logrotate)
sudo tcpdump -i eth0 -n -v -l "ip proto 2" 2>/dev/null | \
  while IFS= read -r line; do
    echo "$(date '+%Y-%m-%dT%H:%M:%S') $line"
  done | tee -a /var/log/igmp-events.log
```

## Watching IGMPv3 Source Filter Changes

IGMPv3 reports carry source lists. Use tshark to decode them:

```bash
sudo tshark -i eth0 -n \
  -Y "igmp.version == 3" \
  -T fields \
  -e frame.time \
  -e ip.src \
  -e igmp.maddr \
  -e igmp.num_grp_recs \
  2>/dev/null
```

## Conclusion

`tcpdump` with `ip proto 2` is the fastest way to watch IGMP events live. For structured output, `tshark` provides field-level decoding. Combine these with a `/proc/net/igmp` polling script to detect membership changes even between IGMP messages.
