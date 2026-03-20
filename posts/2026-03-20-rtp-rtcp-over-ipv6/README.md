# How to Configure RTP/RTCP over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RTP, RTCP, IPv6, VoIP, Media Transport, UDP, Real-Time Protocol

Description: Configure RTP (Real-time Transport Protocol) and RTCP (RTP Control Protocol) media streams over IPv6, including firewall rules, socket binding, and SRTP security.

---

RTP (RFC 3550) carries audio/video media in VoIP calls. Configuring RTP/RTCP over IPv6 involves binding media sockets to IPv6 interfaces, ensuring firewall rules permit UDP port ranges, and updating SDP to advertise IPv6 connection details.

## RTP over IPv6 Architecture

```
RTP/RTCP IPv6 Flow:
Caller (IPv6)                    Callee (IPv6)
2001:db8::caller                 2001:db8::callee

SDP Offer:
c=IN IP6 2001:db8::caller
m=audio 49170 RTP/AVP 0

SDP Answer:
c=IN IP6 2001:db8::callee
m=audio 51372 RTP/AVP 0

Media flow:
2001:db8::caller:49170 <--RTP--> 2001:db8::callee:51372
2001:db8::caller:49171 <--RTCP-> 2001:db8::callee:51373
(RTCP typically uses RTP port + 1)
```

## Python RTP over IPv6 Example

```python
#!/usr/bin/env python3
# rtp_ipv6_sender.py - Send RTP over IPv6

import socket
import struct
import time

def create_rtp_packet(payload, seq, timestamp, ssrc, payload_type=0):
    """Create a minimal RTP packet."""
    # RTP header (12 bytes minimum)
    # V=2, P=0, X=0, CC=0, M=0, PT=payload_type
    byte1 = 0x80  # Version 2
    byte2 = payload_type

    header = struct.pack('!BBHII',
        byte1,          # Version, padding, extension, CSRC count
        byte2,          # Marker bit, payload type
        seq,            # Sequence number
        timestamp,      # Timestamp
        ssrc            # SSRC
    )
    return header + payload

def send_rtp_over_ipv6(dest_ipv6, dest_port, duration_secs=10):
    """Send RTP stream to IPv6 destination."""
    # Create IPv6 UDP socket
    sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
    sock.bind(('::', 0))  # Bind to random port on all IPv6 interfaces

    local_port = sock.getsockname()[1]
    print(f"RTP sender bound to [::]:{local_port}")

    ssrc = 12345678
    seq = 0
    timestamp = 0
    ptime = 20  # 20ms ptime for G.711

    # G.711 silence payload (160 bytes = 20ms at 8000Hz)
    payload = bytes([0xFF] * 160)

    start = time.time()
    while time.time() - start < duration_secs:
        pkt = create_rtp_packet(payload, seq, timestamp, ssrc)
        sock.sendto(pkt, (dest_ipv6, dest_port, 0, 0))  # IPv6 tuple

        seq = (seq + 1) & 0xFFFF
        timestamp += 160  # 160 samples at 8kHz = 20ms

        time.sleep(ptime / 1000.0)

    sock.close()
    print(f"Sent {seq} RTP packets over IPv6")

if __name__ == '__main__':
    # Send RTP to IPv6 destination
    send_rtp_over_ipv6('2001:db8::media-server', 49170, duration_secs=5)
```

## SRTP over IPv6 (Secure RTP)

```bash
# Install libsrtp for SRTP testing
sudo apt install libsrtp2-dev srtp-utils -y

# Test SRTP sender over IPv6
srtp-driver -t sender -a -e 128 \
  -d 2001:db8::receiver:5004

# Verify encrypted RTP packets
sudo tcpdump -i eth0 -nn ip6 and udp and port 5004 -v | head -20
```

## Asterisk RTP Configuration for IPv6

```ini
# /etc/asterisk/rtp.conf

[general]
rtpstart=10000
rtpend=20000

# Bind RTP to all interfaces including IPv6
bindaddr=::

# DSCP marking for RTP
tos=ef
cos=5

# RTCP MUX (RFC 5761)
rtcpmux=yes

# SRTP configuration
# (handled via sdp_crypto or SDES in PJSIP)
```

## Firewall Rules for RTP/RTCP over IPv6

```bash
# Allow RTP/RTCP port range over IPv6
sudo ip6tables -A INPUT -p udp --dport 10000:20000 -j ACCEPT

# Allow SRTP (same ports as RTP but encrypted)
# No separate port range needed

# If using RTCP MUX (single port for RTP+RTCP)
# Only the base RTP port is needed

# For WebRTC DTLS-SRTP
sudo ip6tables -A INPUT -p udp --dport 49152:65535 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6

# Verify RTP ports are open
ss -6 -ulnp | grep -E "1[0-9]{4}"
```

## Testing RTP over IPv6

```bash
# Test RTP socket binding on IPv6
python3 << 'EOF'
import socket
sock = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
sock.bind(('::', 10000))
print(f"RTP socket bound on IPv6: {sock.getsockname()}")
sock.close()
EOF

# Capture RTP traffic to verify IPv6 media flow
sudo tcpdump -i eth0 -nn ip6 and udp and portrange 10000-20000 -v

# Check RTCP statistics (from Asterisk)
asterisk -rx "rtp set debug on"
sudo tail -f /var/log/asterisk/messages | grep "RTCP\|rtp"

# Analyze RTP stream quality
sudo apt install rtptools -y
rtpdump -F ascii -t 10 /0.0.0.0:0 | head -20
```

RTP/RTCP over IPv6 uses IPv6 UDP sockets binding to `::` for the configured port range, with the SDP `c=IN IP6` connection line directing the remote endpoint to send media to the IPv6 address, enabling direct IPv6 media paths that eliminate the NAT traversal complexity common in IPv4 VoIP deployments.
