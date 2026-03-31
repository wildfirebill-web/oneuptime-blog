# How to Use Wireshark Expert Information to Find Network Problems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Wireshark, Expert Information, Networking, Diagnostic, TCP, Performance

Description: Use Wireshark's Expert Information panel to automatically identify network problems including TCP retransmissions, connection resets, malformed packets, and application errors.

The Expert Information panel is Wireshark's automated analysis engine. It scans every packet and flags anomalies by severity level - giving you an instant summary of what's wrong in a capture without manually reading thousands of packets.

## Open Expert Information

```text
Method 1: Analyze → Expert Information

Method 2: Click the colored circle in the bottom-left of Wireshark
  (the circle color reflects the highest severity in the capture)
  Green = normal, Yellow = warning, Red = error

Method 3: When a capture has errors, the bottom status bar shows
  a colored indicator - click it to open Expert Information
```

## Severity Levels

```text
Color      Level     Meaning
---------  --------  ------------------------------------------
Red        Error     Serious problems (retransmissions, RSTs)
Yellow     Warning   Potential issues (duplicate ACKs, out-of-order)
Light blue Note      Normal but noteworthy events (SYN, FIN)
Gray       Chat      Informational (connection events)
```

## Common Expert Information Messages

```yaml
Message                    Severity  Meaning
-----------------------    --------  --------------------------------------
TCP Retransmission         Error     Packet resent = packet loss
Previous segment lost      Error     Gap detected in TCP stream
TCP ACKed unseen segment   Note      Possible capture started mid-stream
Duplicate ACK              Warning   Loss signal, retransmission coming
TCP Fast Retransmission    Error     Loss: 3 dup ACKs received
Out-Of-Order               Warning   Reordering or loss
Connection reset           Error     Unexpected RST
Window Full                Warning   Receiver buffer full (flow control)
Zero Window                Error     Receiver buffer empty (sender paused)
TCP Window Update          Note      Receiver reopening window
Application response time  Warning   Server took too long to respond
DNS NXDOMAIN               Warning   Domain not found
HTTP server error (5xx)    Warning   Application error
```

## Use Expert Information as Starting Point

```text
Workflow:
1. Open capture
2. Open Expert Information
3. Sort by Severity (Error first)
4. Click on an error entry
   → Wireshark jumps to that packet in the main list
5. Examine the packet and surrounding context
   → Use "Follow TCP Stream" for full context
   → Check IO Graphs to see if errors correlate with traffic spikes

Repeat for all Error entries, then Warning entries.
```

## Filter Based on Expert Information

Click "Prepare Filter" in the Expert Information dialog to create a display filter:

```wireshark
# These filters correspond to Expert Information categories

# All TCP errors

tcp.analysis.flags

# Retransmissions
tcp.analysis.retransmission

# Connection resets
tcp.flags.reset == 1

# Window issues
tcp.analysis.zero_window or tcp.analysis.window_full

# All expert info (any flag)
expert
```

## Command-Line Expert Analysis with tshark

```bash
# Print expert information from a PCAP
tshark -r capture.pcap -q -z expert

# Output:
# === Expert Information ===
#
# Errors (5):
#   tcp.analysis.retransmission (5 occurrences)
#
# Warnings (23):
#   tcp.analysis.duplicate_ack (15 occurrences)
#   tcp.analysis.out_of_order (8 occurrences)
#
# Notes (12):
#   tcp.analysis.ack_lost_segment (12 occurrences)

# Quick health check: any errors?
tshark -r capture.pcap -q -z expert | grep -c "Errors"
```

## Export Expert Information

```bash
In Expert Information dialog:
  Right-click → Export to File
  → Saves as CSV or plain text for documentation or ticketing
```

Expert Information should be your first action after opening a capture - it immediately flags the most serious problems, saving you hours of manual packet-by-packet inspection.
