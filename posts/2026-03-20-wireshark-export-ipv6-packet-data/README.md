# How to Export IPv6 Packet Data from Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wireshark, IPv6, Export, tshark, pcap, Data Analysis

Description: A guide to exporting IPv6 packet data from Wireshark in various formats for further analysis, reporting, and integration with other tools.

Wireshark and tshark provide multiple export formats for IPv6 packet data, from filtered pcap files to CSV, JSON, and raw packet bytes. This guide covers all common export methods.

## Export Filtered IPv6 Packets (pcap/pcapng)

### Via Wireshark GUI

1. Apply an IPv6 display filter: `ipv6`
2. Go to **File → Export Specified Packets**
3. In the dialog:
   - Set **Packet Range** to **Displayed** (applies your filter)
   - Choose format: **pcapng** (recommended) or **pcap**
4. Click **Save**

### Via tshark Command Line

```bash
# Export only IPv6 packets from a capture file
tshark -r capture.pcap -Y "ipv6" -w ipv6-only.pcap

# Export IPv6 HTTPS traffic
tshark -r capture.pcap -Y "ipv6 && tcp.port == 443" -w ipv6-https.pcap

# Export IPv6 traffic from a specific source
tshark -r capture.pcap \
  -Y "ipv6.src == 2001:db8::10" \
  -w from-2001db8-10.pcap
```

## Export IPv6 Data to CSV

```bash
# Export IPv6 packet fields to CSV
tshark -r capture.pcap -Y "ipv6" \
  -T fields \
  -e frame.number \
  -e frame.time \
  -e ipv6.src \
  -e ipv6.dst \
  -e ipv6.hlim \
  -e ipv6.plen \
  -e ipv6.nxt \
  -E header=y \
  -E separator=, \
  -E quote=d \
  -E occurrence=f \
  > ipv6-packets.csv
```

## Export to JSON

```bash
# Export IPv6 packets as JSON (one object per packet)
tshark -r capture.pcap -Y "ipv6" \
  -T json \
  > ipv6-packets.json

# Export specific fields as JSON (smaller file)
tshark -r capture.pcap -Y "ipv6" \
  -T jsonraw \
  -e ipv6.src \
  -e ipv6.dst \
  -e frame.time_epoch \
  > ipv6-fields.json
```

## Export ICMPv6 NDP Data

```bash
# Export Neighbor Discovery packets with key fields
tshark -r capture.pcap -Y "icmpv6" \
  -T fields \
  -e frame.time \
  -e ipv6.src \
  -e ipv6.dst \
  -e icmpv6.type \
  -e icmpv6.nd.ns.target_address \
  -e icmpv6.nd.na.target_address \
  -E header=y \
  -E separator=, \
  > ndp-data.csv
```

## Export DHCPv6 Lease Data

```bash
# Export DHCPv6 transactions with assigned addresses
tshark -r capture.pcap -Y "dhcpv6" \
  -T fields \
  -e frame.time \
  -e ipv6.src \
  -e dhcpv6.msgtype \
  -e dhcpv6.iaaddr.ip \
  -e dhcpv6.duidllt.link_layer_addr \
  -E header=y \
  -E separator=, \
  > dhcpv6-leases.csv
```

## Export Raw Packet Bytes

```bash
# Export raw IPv6 packet bytes as hex dump
tshark -r capture.pcap -Y "ipv6" \
  -T fields \
  -e frame.number \
  -e data \
  > ipv6-hex-dump.txt

# Export pcap to hex/text via xxd
tshark -r capture.pcap -Y "ipv6" -w - 2>/dev/null | \
  xxd > ipv6-raw-bytes.hex
```

## Export to XML (PDML)

```bash
# Export to Packet Details Markup Language (XML)
tshark -r capture.pcap -Y "ipv6" \
  -T pdml \
  > ipv6-packets.xml
```

## Batch Export Script

```bash
#!/bin/bash
# export-ipv6-analysis.sh - Export IPv6 data in multiple formats

INPUT_PCAP="$1"
BASE="${INPUT_PCAP%.pcap}"

echo "Exporting IPv6 data from $INPUT_PCAP..."

# Filtered pcap
tshark -r "$INPUT_PCAP" -Y "ipv6" -w "${BASE}-ipv6.pcap"

# CSV summary
tshark -r "$INPUT_PCAP" -Y "ipv6" \
  -T fields -e frame.time -e ipv6.src -e ipv6.dst \
  -e frame.len -e ipv6.nxt \
  -E header=y -E separator=, > "${BASE}-ipv6-summary.csv"

# Conversation stats
tshark -r "$INPUT_PCAP" -q -z conv,ipv6 > "${BASE}-ipv6-conversations.txt"

echo "Exports complete:"
ls -lh "${BASE}-ipv6"*
```

## Run the batch export

```bash
chmod +x export-ipv6-analysis.sh
./export-ipv6-analysis.sh network-capture.pcap
```

Wireshark and tshark's export capabilities allow IPv6 packet data to be ingested by log analysis tools, ELK/Splunk stacks, Python analysis scripts, and security information platforms for correlation and long-term storage.
