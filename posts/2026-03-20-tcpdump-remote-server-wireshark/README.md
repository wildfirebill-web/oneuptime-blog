# How to Capture Packets on a Remote Server with tcpdump and Analyze in Wireshark

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: tcpdump, Remote Capture, Wireshark, SSH, Packet Analysis

Description: Learn how to capture packets on a remote Linux server using tcpdump over SSH and stream the capture directly to Wireshark on your local machine for real-time analysis without storing PCAP files.

## The Remote Capture Challenge

When troubleshooting network issues on a remote server, you need to capture packets where the traffic flows — not on your laptop. The solution is to run tcpdump on the remote server and pipe the capture through SSH to Wireshark on your local machine.

## Step 1: Stream Remote Capture to Local Wireshark

```bash
# Run tcpdump on remote server, pipe through SSH to local Wireshark
# This streams live packets without writing files on the server

# Basic remote capture
ssh user@remote-server "sudo tcpdump -i eth0 -n -w - 'not port 22'" | wireshark -k -i -

# Explanation:
# -w -         : Write PCAP to stdout (not a file)
# 'not port 22': Exclude SSH traffic (prevents capture feedback loop)
# | wireshark -k -i -  : Open Wireshark, start capturing from stdin (-k), input is stdin (-i -)
```

```bash
# With specific filter on remote
ssh user@192.168.1.100 "sudo tcpdump -i eth0 -n -w - 'port 80 or port 443'" | wireshark -k -i -

# Capture from specific interface on remote
ssh user@192.168.1.100 "sudo tcpdump -i ens3 -n -w - 'host 10.0.0.50'" | wireshark -k -i -
```

## Step 2: Capture and Save Locally

```bash
# Stream remote capture and save to local PCAP file
ssh user@remote-server "sudo tcpdump -i eth0 -n -w - -c 1000 'not port 22'" \
    > /tmp/remote-capture.pcap

# Open the saved file
wireshark /tmp/remote-capture.pcap

# Capture for a specific duration
ssh user@remote-server "sudo timeout 30 tcpdump -i eth0 -n -w - 'port 8080'" \
    > /tmp/remote-30s.pcap
```

## Step 3: Use SSH Config for Convenience

```bash
# ~/.ssh/config — set up reusable server config
cat >> ~/.ssh/config << 'EOF'
Host myserver
    HostName 192.168.1.100
    User admin
    IdentityFile ~/.ssh/id_rsa
    RequestTTY no        # Important: no TTY for piping binary data
EOF

# Now capture with shorthand
ssh myserver "sudo tcpdump -i eth0 -n -w - 'not port 22'" | wireshark -k -i -
```

## Step 4: Handle Authentication Issues

```bash
# If tcpdump requires sudo with password prompt (breaks pipe)
# Solution 1: Add tcpdump to sudoers with NOPASSWD

# On remote server:
sudo visudo
# Add: admin ALL=(ALL) NOPASSWD: /usr/sbin/tcpdump

# Solution 2: Use SSH key authentication (no password prompt)
ssh-copy-id user@remote-server

# Solution 3: Run tcpdump with capabilities instead of sudo
sudo setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump
# Then non-root can run: tcpdump -i eth0 -w - 'port 80'
```

## Step 5: Capture on Multiple Remote Interfaces

```bash
# Capture on all interfaces of remote server
ssh user@remote-server "sudo tcpdump -i any -n -w - 'not port 22'" | wireshark -k -i -

# Capture only IPv4 traffic
ssh user@remote-server "sudo tcpdump -i eth0 -n -w - 'ip and not port 22'" | wireshark -k -i -

# Higher packet rate — increase buffer
ssh user@remote-server "sudo tcpdump -i eth0 -n -B 65536 -w - 'not port 22'" | wireshark -k -i -
# -B 65536 = 64MB kernel buffer to reduce drops
```

## Step 6: Automated Remote Capture with Trigger

```bash
#!/bin/bash
# capture-on-remote.sh — start capture when issue detected

REMOTE="user@192.168.1.100"
LOCAL_DIR="/tmp/captures"
mkdir -p "$LOCAL_DIR"

TIMESTAMP=$(date +%Y%m%d-%H%M%S)
PCAP_FILE="$LOCAL_DIR/capture-$TIMESTAMP.pcap"

echo "Starting capture on remote server..."
echo "Saving to: $PCAP_FILE"
echo "Press Ctrl+C to stop"

# Capture until interrupted, save locally
ssh "$REMOTE" "sudo tcpdump -i eth0 -n -w - 'not port 22'" > "$PCAP_FILE"

echo "Capture complete. File: $PCAP_FILE"
echo "Opening in Wireshark..."
wireshark "$PCAP_FILE" &
```

## Step 7: Use tshark Instead of Wireshark (Headless)

```bash
# If no GUI is available, use tshark on local side
ssh user@remote-server "sudo tcpdump -i eth0 -n -w - 'port 443'" | \
    tshark -r - -T fields -e ip.src -e ip.dst -e tcp.flags

# Extract specific fields from remote capture
ssh user@remote-server "sudo tcpdump -i eth0 -n -w - 'port 80'" | \
    tshark -r - -T fields -e http.request.method -e http.request.uri -e ip.src

# Count packets by protocol
ssh user@remote-server "sudo tcpdump -i eth0 -n -w - -c 1000 'not port 22'" | \
    tshark -r - -q -z io,phs
```

## Conclusion

Remote packet capture to Wireshark uses: `ssh user@server "sudo tcpdump -i eth0 -n -w - 'not port 22'" | wireshark -k -i -`. The key elements are `-w -` (write to stdout), `not port 22` (exclude SSH to prevent feedback), and `wireshark -k -i -` (start immediately, read from stdin). For headless analysis, pipe to `tshark -r -` instead. Set up passwordless sudo for tcpdump on the remote server to avoid password prompts breaking the binary pipe.
