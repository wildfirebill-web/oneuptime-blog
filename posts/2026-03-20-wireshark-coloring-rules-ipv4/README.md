# How to Use Wireshark Coloring Rules for IPv4 Traffic Analysis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wireshark, IPv4, Networking, Packet Analysis, Troubleshooting, Network Diagnostics

Description: Learn how to use and create Wireshark coloring rules to visually differentiate IPv4 traffic types, making it faster to identify issues during packet captures.

## Introduction

Wireshark's coloring rules apply visual highlights to packets based on filters, making it much easier to spot patterns, errors, and specific traffic types during analysis. This guide covers the built-in IPv4 coloring rules and how to create custom ones for your specific needs.

## Built-In IPv4 Coloring Rules

Wireshark ships with default coloring rules for common conditions. Navigate to **View → Coloring Rules** to see and manage them.

Common default rules:
- Red: TCP RST (connection resets)
- Dark red: TCP errors, bad checksums
- Green: HTTP traffic
- Light blue: TCP traffic
- Dark blue: DNS traffic

## Viewing and Editing Coloring Rules

1. Open Wireshark
2. Go to **View → Coloring Rules**
3. Click **+** to add a new rule
4. Enter a name, filter expression, and background/foreground colors

## Useful IPv4 Coloring Rule Filters

**Highlight ICMP traffic:**

```text
icmp
```

**Highlight traffic from a specific subnet:**

```text
ip.src == 192.168.1.0/24
```

**Highlight high-latency TCP (retransmissions):**

```text
tcp.analysis.retransmission || tcp.analysis.fast_retransmission
```

**Highlight TCP connection resets:**

```text
tcp.flags.reset == 1
```

**Highlight multicast IPv4:**

```text
ip.dst >= 224.0.0.0 and ip.dst <= 239.255.255.255
```

**Highlight fragmented IPv4 packets:**

```text
ip.flags.mf == 1 || ip.frag_offset > 0
```

**Highlight duplicate ACKs:**

```text
tcp.analysis.duplicate_ack
```

## Creating a Custom Color Rule

Example: Highlight traffic to a specific server in yellow:

1. Go to **View → Coloring Rules → +**
2. Name: `Server Traffic`
3. Filter: `ip.addr == 10.0.1.50`
4. Background: yellow
5. Foreground: black
6. Click **OK**

## Exporting and Importing Color Rules

Export your rules to share with your team:

```bash
# Rules are stored at:

~/.config/wireshark/colorfilters
```

Import: **View → Coloring Rules → Import**

## Using Display Filters with Colors

Apply a display filter to reduce noise, then use colors to categorize what remains:

```text
# Display filter: only show one subnet
ip.addr == 10.0.0.0/8

# Then color rules apply to the filtered view
```

## Temporary Conversation Coloring

Right-click a packet → **Colorize Conversation** → choose a color. This is temporary and doesn't save to coloring rules.

## Conclusion

Wireshark coloring rules transform raw packet captures into visually organized traffic views where issues like retransmissions, resets, and specific host traffic stand out immediately. Creating a custom ruleset for your environment accelerates troubleshooting and helps you spot anomalies at a glance.
