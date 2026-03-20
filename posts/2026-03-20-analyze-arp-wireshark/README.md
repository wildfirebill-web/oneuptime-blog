# How to Analyze ARP Traffic with Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, ARP, Wireshark, Packet Analysis

Description: Learn how to capture and analyze ARP packets in Wireshark to troubleshoot ARP issues and detect anomalies.

## Capturing ARP Traffic in Wireshark

### Method 1: Capture Filter (Before Capture)

Set a capture filter to record only ARP packets:

```
arp
```

In Wireshark: **Capture → Options → Enter filter: `arp`**

### Method 2: Display Filter (After Capture)

Filter ARP from a full capture:

```
arp
```

## Understanding the Wireshark ARP Packet View

A typical ARP request in Wireshark shows:

```
Frame: 60 bytes on wire
Ethernet II:
  Destination: Broadcast (ff:ff:ff:ff:ff:ff)
  Source: aa:bb:cc:dd:ee:01 (Vendor Name)
  Type: ARP (0x0806)

Address Resolution Protocol (request):
  Hardware type: Ethernet (1)
  Protocol type: IPv4 (0x0800)
  Hardware size: 6
  Protocol size: 4
  Opcode: request (1)
  Sender MAC address: aa:bb:cc:dd:ee:01
  Sender IP address: 192.168.1.10
  Target MAC address: 00:00:00:00:00:00
  Target IP address: 192.168.1.20
```

## Useful Wireshark Display Filters

```
# Show all ARP packets
arp

# Show only ARP requests
arp.opcode == 1

# Show only ARP replies
arp.opcode == 2

# Show gratuitous ARP (sender IP = target IP)
arp.isgratuitous == true

# Show ARP for a specific IP
arp.src.proto_ipv4 == 192.168.1.10

# Show ARP involving a specific MAC
arp.src.hw_mac == aa:bb:cc:dd:ee:01

# Detect possible ARP spoofing (same IP, different MACs)
arp.duplicate-address-detected
```

## Identifying ARP Spoofing in Wireshark

Wireshark automatically highlights potential ARP spoofing with a warning:

- **"Duplicate IP address detected"** — appears when two different MACs claim the same IP
- Color-coded in red/yellow in the packet list

Filter for duplicate IP detection:

```
arp.duplicate-address-detected || arp.duplicate-address-frame
```

## Building an ARP Summary with Wireshark Statistics

Go to **Statistics → Endpoints** to see all Layer 2 (Ethernet) addresses that sent ARP, with packet counts.

Or use **Statistics → Protocol Hierarchy** to see what percentage of traffic is ARP.

## Exporting ARP Data via tshark

```bash
# Capture ARP and output fields
tshark -i eth0 -f "arp" -T fields \
  -e arp.opcode \
  -e arp.src.proto_ipv4 \
  -e arp.src.hw_mac \
  -e arp.dst.proto_ipv4 \
  -e arp.dst.hw_mac

# Sample output:
# 1  192.168.1.10  aa:bb:cc:dd:ee:01  192.168.1.20  00:00:00:00:00:00
# 2  192.168.1.20  00:11:22:33:44:55  192.168.1.10  aa:bb:cc:dd:ee:01
```

## Detecting ARP Storms

An ARP storm shows a very high rate of ARP requests from one or more sources:

```bash
# Use tshark to count ARP packets per second
tshark -i eth0 -f "arp" -a duration:10 -q -z io,stat,1
```

## Key Takeaways

- Use `arp` as both capture and display filter in Wireshark.
- `arp.opcode == 1` filters requests, `arp.opcode == 2` filters replies.
- Wireshark automatically detects gratuitous ARPs with `arp.isgratuitous`.
- The "duplicate address detected" warning in Wireshark can indicate ARP spoofing.

**Related Reading:**

- [How to Capture ARP Packets with tcpdump](https://oneuptime.com/blog/post/2026-03-20-capture-arp-tcpdump/view)
- [How to Detect ARP Spoofing Attacks on Your Network](https://oneuptime.com/blog/post/2026-03-20-arp-spoofing-detection-scapy-ipv4/view)
- [How to Troubleshoot ARP Resolution Failures](https://oneuptime.com/blog/post/2026-03-20-troubleshoot-arp-resolution/view)
