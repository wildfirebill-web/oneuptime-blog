# How to Diagnose and Fix TCP Connection Resets (RST Packets)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP RST, Connection Reset, Wireshark, tcpdump, Troubleshooting

Description: Learn how to diagnose TCP connection resets by capturing and analyzing RST packets with tcpdump and Wireshark, then identify whether the cause is firewall rules, application errors, timewall...

## What Causes TCP RST Packets?

RST (Reset) packets immediately abort a TCP connection. Sources include:
- **Application crash**: Server process dies mid-connection
- **Firewall reject**: Stateful firewall drops packet and sends RST
- **Load balancer timeout**: Idle connections terminated after timeout
- **TCP keepalive failure**: OS closes idle socket
- **NAT table expiry**: Connection entry removed, next packet triggers RST
- **Port scan response**: Target port is closed (not listening)

## Step 1: Capture RST Packets

```bash
# Capture all TCP RST packets

sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0' -v -n

# Example RST output:
# 12:05:01.234 IP 192.168.1.100.80 > 10.0.0.50.54321: Flags [R.], seq 0, ack 1, win 0
# "Flags [R.]" = RST+ACK
# "Flags [R]"  = RST only

# Capture RSTs for a specific connection
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0 and host 192.168.1.100' -v

# Save to file for Wireshark analysis
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0' -w /tmp/rst-capture.pcap
```

## Step 2: Identify the Source of RSTs

```bash
# Count RSTs by source IP - which device is sending them?
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0' -n 2>/dev/null | \
    awk '{print $3}' | sed 's/\.[0-9]*$//' | sort | uniq -c | sort -rn | head -10

# If RSTs come FROM the server:
# → Application closed connection, firewall sending RST, or OS timeout

# If RSTs come FROM the client:
# → Client received unexpected data (e.g., load balancer re-used session)

# If RSTs come FROM a middle device (different IP from server/client):
# → Firewall or IDS/IPS is intercepting and sending RSTs
```

## Step 3: Wireshark Analysis

```text
Wireshark filters for RST analysis:

1. All RST packets:
   tcp.flags.reset == 1

2. RST in established connections (not port-closed responses):
   tcp.flags.reset == 1 and tcp.seq > 1

3. Show complete stream around the RST:
   Follow → TCP Stream (right-click on RST packet)

4. Find RST after idle period:
   tcp.flags.reset == 1 and tcp.analysis.idle_time > 60

Statistics → Conversations → TCP
Shows connection duration - very short durations suggest RST issues
```

## Step 4: Check Firewall for RST Injection

```bash
# iptables - check for REJECT rules (send RST to TCP)
sudo iptables -L -n | grep -i "reject\|rst"

# REJECT --reject-with tcp-reset sends RST to client
# REJECT --reject-with icmp-port-unreachable sends ICMP

# Check conntrack for expired states
sudo conntrack -L | grep ESTABLISHED | wc -l
sudo conntrack -L | grep TIME_WAIT | wc -l

# If many TIME_WAIT entries, connections are being closed normally
# If established connections are dropping suddenly: firewall timeout issue
```

## Step 5: Fix Idle Connection Timeouts

```bash
# Linux - TCP keepalive settings
# Keepalive probes prevent NAT/firewall from expiring idle connections

sysctl net.ipv4.tcp_keepalive_time     # 7200 = 2 hours (too long)
sysctl net.ipv4.tcp_keepalive_intvl   # 75 seconds between probes
sysctl net.ipv4.tcp_keepalive_probes  # 9 probes before RST

# Reduce keepalive time to detect failures sooner
sudo tee -a /etc/sysctl.conf << 'EOF'
net.ipv4.tcp_keepalive_time = 300     # Send first keepalive after 5 min idle
net.ipv4.tcp_keepalive_intvl = 30    # Then every 30 seconds
net.ipv4.tcp_keepalive_probes = 3    # Give up after 3 missed probes
EOF
sudo sysctl -p
```

## Step 6: Fix Application-Level RSTs

```python
# Python - handle RST gracefully and reconnect
import socket
import time

def connect_with_retry(host, port, retries=3, delay=2):
    """Connect with automatic retry on RST."""
    for attempt in range(retries):
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
            sock.connect((host, port))
            return sock
        except ConnectionResetError as e:
            print(f"Connection reset on attempt {attempt + 1}: {e}")
            if attempt < retries - 1:
                time.sleep(delay)
    raise ConnectionError(f"Failed after {retries} attempts")

# Enable SO_KEEPALIVE to prevent NAT timeout RSTs
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
```

## Step 7: Monitor RST Rate Over Time

```bash
# Watch RST rate using nstat
nstat -z | grep TcpAttemptFails    # Connection attempts that got RST
nstat -z | grep TcpEstabResets     # Established connections that got RST

# High TcpEstabResets = existing connections being forcefully closed
# High TcpAttemptFails = RSTs on SYN packets (port closed or firewall)

# Monitor with ss
ss -tan | awk '{print $1}' | sort | uniq -c
# Watch TIME-WAIT count - should stabilize, not grow indefinitely
```

## Conclusion

TCP RSTs are captured with `tcpdump 'tcp[tcpflags] & tcp-rst != 0'` and analyzed in Wireshark using the `tcp.flags.reset == 1` filter. Identify the RST source: if from a third IP, a firewall/IDS is injecting RSTs. Fix idle timeout RSTs by reducing `tcp_keepalive_time` to 300 seconds. Use `nstat | grep TcpEstabResets` to monitor whether established connections are being reset by the application or network infrastructure. Follow TCP streams in Wireshark to see what data preceded each RST.
