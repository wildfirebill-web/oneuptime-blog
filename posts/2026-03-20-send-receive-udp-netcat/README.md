# How to Send and Receive UDP Packets on Linux with netcat

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: UDP, netcat, Linux, Networking, Testing, Socket

Description: Use netcat (nc) to send and receive UDP packets on Linux for testing UDP services, verifying port availability, and debugging UDP-based protocols.

## Introduction

`netcat` is the TCP/IP Swiss Army knife, and its UDP mode is just as useful as TCP mode. You can use it to test whether a UDP port is open, send test payloads to UDP services, build simple UDP echo servers for debugging, and verify that UDP traffic is flowing through firewalls and NAT correctly. Unlike TCP, there is no connection state, so netcat exits or continues sending based on flags you provide.

## Basic UDP Send and Receive

```bash
# Terminal 1: Listen for UDP on port 5000

nc -ul 5000
# -u: UDP mode
# -l: listen mode

# Terminal 2: Send a UDP packet
echo "hello udp" | nc -u 127.0.0.1 5000
# Sends "hello udp\n" to localhost:5000 UDP

# Terminal 1 will print: hello udp
```

## Persistent UDP Listener

```bash
# By default, nc in UDP listen mode exits after first message
# For persistent listener (keep running):
while true; do
    nc -ul 5000
    echo "[nc restarted, waiting...]"
done

# Or use nc -k (keep-alive, available in GNU netcat):
nc -ulk 5000  # GNU netcat (Ubuntu, Debian)

# Check which netcat you have:
nc --version 2>&1 | head -1
# OpenBSD netcat: -k may not work
# GNU netcat: -k supported
```

## UDP Echo Test

```bash
# Create a UDP echo server (receives and sends back)
nc -ul 5000 | nc -u 127.0.0.1 5000
# Pipe: anything received on 5000, send back to 5000

# Simpler echo server using socat (more reliable for UDP):
socat UDP4-LISTEN:5000,fork PIPE

# Test it:
echo "ping" | nc -u 127.0.0.1 5000
```

## Testing UDP Port Availability

```bash
# Send a UDP probe to test if port is reachable
# Note: UDP gives no confirmation unless the service responds

# Method 1: Send data and wait for response
echo "test" | nc -u -w 2 10.20.0.5 53
# -w 2: wait 2 seconds for response
# If nothing: port may be open (service didn't respond) or closed (ICMP unreachable)

# Method 2: Check for ICMP port unreachable (port closed)
# Open a listener with ncat that shows verbose output:
ncat -u -l 5000 -v

# Method 3: nmap UDP scan (most reliable)
nmap -sU -p 53 10.20.0.5
# Requires root; actually sends probes and checks for ICMP unreachable
```

## Sending Specific Payloads

```bash
# Send hex payload
printf '\x00\x0a\x01\x00' | nc -u 10.20.0.5 5000

# Send a DNS query (raw):
# This queries for "google.com" A record
python3 -c "
import socket
# Minimal DNS query for google.com type A
query = b'\x00\x01\x01\x00\x00\x01\x00\x00\x00\x00\x00\x00'
query += b'\x06google\x03com\x00'
query += b'\x00\x01\x00\x01'
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.sendto(query, ('8.8.8.8', 53))
data, _ = s.recvfrom(512)
print('Got response:', len(data), 'bytes')
"

# Send a file over UDP (unreliable, may be truncated at MTU)
nc -u 10.20.0.5 5000 < /tmp/testfile
# Note: large files will be fragmented or dropped; UDP is not for large data without application framing
```

## UDP with Timeout

```bash
# Send and wait for response with timeout
echo "query" | nc -u -w 3 10.20.0.5 5000
# -w 3: exit 3 seconds after last activity

# For scripted testing:
if echo "test" | nc -u -w 2 10.20.0.5 5000 | grep -q "expected_response"; then
    echo "Service responded correctly"
else
    echo "Service unavailable or wrong response"
fi
```

## Debugging UDP Traffic Flow

```bash
# In one terminal: capture UDP traffic
tcpdump -i eth0 -n 'udp port 5000'

# In another: send test packets
for i in $(seq 1 5); do
    echo "packet $i" | nc -u 10.20.0.5 5000
    sleep 0.5
done

# Verify each packet appears in tcpdump output
# If packets appear at sender but not receiver: firewall or routing issue
# If packets appear at receiver but app doesn't respond: application issue
```

## Conclusion

`netcat` in UDP mode (`-u`) is the fastest way to verify UDP connectivity. Send a test packet with `echo | nc -u host port`, listen with `nc -ul port`, and combine with `tcpdump` to see exactly where packets reach. Remember: unlike TCP, there is no confirmation of delivery in UDP itself - you only know a packet arrived if the application responds. ICMP port unreachable is the only automatic signal that a closed UDP port provides.
