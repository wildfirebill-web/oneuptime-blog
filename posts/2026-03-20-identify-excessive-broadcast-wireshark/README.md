# How to Identify Excessive Broadcast Traffic with Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Wireshark, Broadcast, Network Analysis, Troubleshooting, Packet Capture

Description: Use Wireshark's display filters, statistics, and IO graphs to identify excessive broadcast traffic, find the top senders, and diagnose the root protocol causing the flood.

## Introduction

Wireshark is the most powerful tool for deep-diving into broadcast traffic. Its statistics and filtering capabilities let you quantify broadcast volume, identify the noisiest senders, and decode the exact protocol generating the flood.

## Capturing Broadcast Traffic

In Wireshark's capture filter field, enter:

```text
ether broadcast
```

Or, to also capture multicast:

```text
ether multicast
```

This limits the capture to only broadcast and multicast frames, reducing capture file size significantly.

## Display Filters for Broadcasts

Once captured, use these display filters in the filter bar:

```text
# All Ethernet broadcasts

eth.dst == ff:ff:ff:ff:ff:ff

# ARP broadcasts only
arp

# DHCP broadcasts
bootp

# NetBIOS name service broadcasts
nbns

# mDNS
mdns

# SSDP
ssdp
```

## Finding the Top Broadcast Senders

Navigate to **Statistics > Endpoints** and click the **Ethernet** tab. Sort by **Packets** in the **TX** column - the highest senders are your broadcast sources.

For IP-level breakdown, check **Statistics > Conversations > IPv4** and filter for destination `255.255.255.255`.

## Using IO Graphs to Spot Storms

Navigate to **Statistics > IO Graphs**:

1. Add a trace for `eth.dst == ff:ff:ff:ff:ff:ff` with **Packets/s** on the Y-axis
2. Set the interval to 1 second
3. Look for sudden spikes indicating storm onset

## Protocol Breakdown with Statistics > Protocol Hierarchy

Navigate to **Statistics > Protocol Hierarchy** to see which protocols account for the most packets. In a broadcast storm, one protocol (often ARP or NetBIOS) will dominate the hierarchy.

## Finding the Storm Source

Apply a display filter for the top offending protocol:

```text
arp
```

Then navigate to **Statistics > Conversations > Ethernet**. The source MAC sending thousands of ARP requests is the culprit.

To confirm it is gratuitous or anomalous:

```text
arp.opcode == 1 && arp.src.proto_ipv4 == arp.dst.proto_ipv4
```

This filter matches **gratuitous ARP requests** - a host announcing its own IP, which could indicate IP address conflicts or a loop.

## Exporting Broadcast Statistics to CSV

For reporting, export statistics:

1. **Statistics > Endpoints** → select Ethernet → click **Copy** → paste to spreadsheet

Or use `tshark` from the command line:

```bash
# Count broadcast packets per source MAC in a pcap file
tshark -r capture.pcap \
  -Y "eth.dst == ff:ff:ff:ff:ff:ff" \
  -T fields -e eth.src \
  | sort | uniq -c | sort -rn | head -20
```

## Conclusion

Wireshark's **IO Graphs**, **Statistics > Endpoints**, and **Protocol Hierarchy** provide a complete picture of broadcast traffic. Capture with `ether broadcast` to reduce noise, then use display filters and statistics to find the top senders and the protocol generating the flood.
