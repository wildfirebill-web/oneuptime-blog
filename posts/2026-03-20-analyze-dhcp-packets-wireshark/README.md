# How to Analyze DHCP Packets in Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Wireshark, Packet Analysis, Network Diagnostics

Description: Wireshark provides detailed DHCP packet dissection showing all options, message types, and field values, enabling engineers to diagnose lease failures, verify option delivery, and investigate rogue servers.

## Capturing DHCP Traffic

Start a capture in Wireshark on your network interface. Apply a capture filter before starting:

```
# Capture filter (BPF syntax)
port 67 or port 68
```

## Display Filters for DHCP

```
# Show all DHCP packets
bootp

# Filter by message type
bootp.option.dhcp == 1   # DHCPDISCOVER
bootp.option.dhcp == 2   # DHCPOFFER
bootp.option.dhcp == 3   # DHCPREQUEST
bootp.option.dhcp == 5   # DHCPACK
bootp.option.dhcp == 6   # DHCPNAK

# Filter by client MAC
bootp.hw.mac_addr == aa:bb:cc:dd:ee:ff

# Filter by offered IP
bootp.ip.your == 192.168.1.105

# Show packets with specific option (e.g., option 43)
bootp.option.type == 43
```

## Reading the DHCP Dissection

When you click a DHCP packet in Wireshark and expand "Bootstrap Protocol (DHCP)":

- **Message type**: 1=Boot Request, 2=Boot Reply
- **Client IP address**: 0.0.0.0 for new requests
- **Your (client) IP address**: Offered/Assigned IP
- **Next server IP address**: TFTP server (option 66/siaddr)
- **Client MAC address**: Hardware address of client
- **Options section**:
  - Option 53: DHCP Message Type
  - Option 54: DHCP Server Identifier
  - Option 51: Lease Time
  - Option 1: Subnet Mask
  - Option 3: Router
  - Option 6: Domain Name Server

## tshark CLI Equivalent

```bash
# Show DHCP message type and key fields for each packet
tshark -r capture.pcap -Y "bootp" \
    -T fields \
    -e frame.number \
    -e ip.src \
    -e ip.dst \
    -e bootp.option.dhcp \
    -e bootp.ip.your \
    -e bootp.option.router \
    -e bootp.option.domain_name_server \
    -E header=y -E separator=,

# Statistics: count each DHCP message type
tshark -r capture.pcap -q -z bootp,stat
```

## Following a Complete DORA Conversation

In Wireshark:
1. Click any DHCP packet from your client.
2. Right-click → **Follow** → **UDP Stream**.
3. All four DORA messages for that client will be shown in sequence.

Or filter by transaction ID:
```
bootp.id == 0x12345678
```

## Identifying Issues in Wireshark

| Observation | Diagnosis |
|-------------|-----------|
| Discovers only (no offers) | DHCP server unreachable |
| NAK message | Client's IP invalid for current network |
| Multiple offers from different IPs | Rogue DHCP server |
| Offer with wrong gateway | Misconfigured server |
| Missing option 3 (router) | No default gateway delivered |

## Key Takeaways

- Use `bootp` as the display filter in Wireshark for all DHCP traffic.
- Click the Options section to verify every DHCP option value delivered to the client.
- `tshark -q -z bootp,stat` provides a quick count of each DHCP message type in a capture.
- Filter by `bootp.hw.mac_addr` to isolate a single client's complete DHCP conversation.
