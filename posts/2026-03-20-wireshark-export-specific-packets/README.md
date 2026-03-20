# How to Export Specific Packets from a Wireshark Capture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wireshark, PCAP, Export, Networking, Packet Analysis

Description: Export filtered or selected packets from a Wireshark capture to a new PCAP file for sharing with colleagues, reducing file size, or focusing follow-up analysis.

Large captures containing millions of packets are impractical to share or analyze. Exporting only the relevant subset - filtered by IP, time range, or manually selected - creates focused PCAP files that are manageable and targeted.

## Export All Currently Displayed Packets

After applying a display filter, export only what's visible:

```bash
1. Apply a display filter in the filter bar
   Example: ip.addr == 192.168.1.100 and tcp.port == 443

2. File → Export Specified Packets

3. In the dialog:
   - Choose "Displayed" (packets matching current filter)
   - Set output format: pcap or pcapng
   - Enter filename
   - Click Save

Result: new PCAP with only packets matching your filter
```

## Export a Manual Selection

Export specific packets you've manually marked:

```sql
1. Click a packet in the packet list to select it
2. Ctrl+click to select additional packets
3. Shift+click to select a range

4. File → Export Specified Packets
   - Choose "Selected" instead of "Displayed"
   - Save

Tip: use Ctrl+A to select all packets in the current display
then switch to "Displayed" for filter-based export
```

## Export by Packet Range

Export packets N through M by number:

```bash
File → Export Specified Packets
  Select: "Packet Range"
  Enter: start and end packet numbers (e.g., 100 to 500)
  Save
```

## Export Marked Packets

Mark specific packets of interest with Ctrl+M, then export them:

```bash
1. Find an interesting packet
2. Press Ctrl+M (or Edit → Mark Packet)
   → Packet row turns black/highlighted

3. Mark multiple packets of interest

4. File → Export Specified Packets
   - Choose "Marked"
   - Save
```

## Extract Files from HTTP Captures

Wireshark can reassemble and export files transferred over HTTP:

```sql
1. Capture HTTP (not HTTPS - must be unencrypted)
2. File → Export Objects → HTTP

Shows list of all HTTP objects (images, HTML, JS, files)
  - Click to select individual files
  - "Save All" to export all
  - "Save" to export selected

Also available:
  File → Export Objects → DICOM (medical imaging)
  File → Export Objects → FTP-DATA
  File → Export Objects → TFTP
```

## Command-Line Export with tshark

For automation or scripting:

```bash
# Export filtered packets using tshark

tshark -r /tmp/large-capture.pcap \
  -Y 'ip.addr == 192.168.1.100' \
  -w /tmp/host-traffic.pcap

# Export by time range (first 60 seconds)
tshark -r /tmp/capture.pcap \
  -Y 'frame.time_relative <= 60' \
  -w /tmp/first-minute.pcap

# Export with multiple filters
tshark -r /tmp/capture.pcap \
  -Y 'tcp.analysis.retransmission or dns.flags.rcode != 0' \
  -w /tmp/problems-only.pcap

# Check exported file size
ls -lh /tmp/problems-only.pcap
```

## Merge Exported Files

```bash
# Merge two PCAP files (requires wireshark-common / mergecap)
mergecap -w /tmp/merged.pcap \
  /tmp/export1.pcap /tmp/export2.pcap

# Merge and sort by timestamp
mergecap -w /tmp/merged.pcap -F pcap \
  /tmp/*.pcap
```

Exporting specific packets is essential for collaboration - instead of sharing a 10 GB capture file, you share a 10 MB file containing only the evidence of the specific problem you're investigating.
