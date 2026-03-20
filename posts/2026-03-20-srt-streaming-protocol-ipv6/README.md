# How to Configure SRT Streaming Protocol with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SRT, IPv6, Streaming, Low-Latency, Live Streaming, FFmpeg, Media Transport

Description: Configure Secure Reliable Transport (SRT) protocol for low-latency live streaming over IPv6, including sender/receiver setup and firewall configuration.

---

SRT (Secure Reliable Transport) is an open-source streaming protocol optimized for low-latency, error-resilient transmission over unstable networks. SRT supports IPv6 natively and can be used for both live ingest and delivery.

## Installing SRT

```bash
# Ubuntu/Debian
sudo apt install libsrt-dev srt-tools -y

# Or build from source
git clone https://github.com/Haivision/srt.git
cd srt
mkdir _build && cd _build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local
make -j4
sudo make install

# Verify installation
srt-live-transmit --version

# Install FFmpeg with SRT support
sudo apt install ffmpeg -y
ffmpeg -protocols | grep srt
```

## SRT over IPv6 - Basic Setup

```bash
# SRT Receiver (listener mode on IPv6)
# Listen on all IPv6 interfaces
srt-live-transmit "srt://:9000" "file://con"

# Listen on specific IPv6 address
srt-live-transmit "srt://[2001:db8::server]:9000" "file://con"

# SRT Sender (caller mode connecting to IPv6 receiver)
srt-live-transmit "file://video.ts" "srt://[2001:db8::server]:9000"

# SRT with options over IPv6
srt-live-transmit \
  "srt://[2001:db8::server]:9000?latency=200&encryption=1&passphrase=SecurePass" \
  "file://con"
```

## SRT Streaming with FFmpeg over IPv6

```bash
# Send stream to IPv6 SRT receiver
ffmpeg -re -i input.mp4 \
  -c:v libx264 \
  -c:a aac \
  -f mpegts \
  "srt://[2001:db8::server]:9000?mode=caller"

# Receive SRT stream from IPv6 sender and convert to HLS
ffmpeg -i "srt://[::]:9000?mode=listener" \
  -c:v copy \
  -c:a copy \
  -f hls \
  /var/www/html/hls/stream.m3u8

# SRT rendezvous mode (both sides connect simultaneously)
# Useful for NAT traversal between IPv6 endpoints
ffmpeg -i input.ts \
  -f mpegts \
  "srt://[2001:db8::peer]:9000?mode=rendezvous"
```

## SRT Relay with srt-live-transmit

```bash
# Relay SRT from IPv4 source to IPv6 destination
srt-live-transmit \
  "srt://source-server:9000" \
  "srt://[2001:db8::destination]:9001"

# Bidirectional relay (both IPv4 and IPv6 endpoints)
srt-live-transmit \
  "srt://[::]:9000?mode=listener" \
  "srt://[2001:db8::encoder]:9001"
```

## SRT Server with srtla (SRT Link Aggregation)

```bash
# Install srtla for multi-path SRT
# Useful for mobile streaming with both IPv4 and IPv6 uplinks
git clone https://github.com/BELABOX/srtla.git
cd srtla && make

# Start srtla receiver (accepts IPv4 and IPv6)
./srtla_rec 9000 [::1] 9001

# Start SRT server listening for relay
srt-live-transmit "srt://[::1]:9001" "file://con"
```

## Firewall Rules for SRT over IPv6

```bash
# Allow SRT port (UDP) over IPv6
sudo ip6tables -A INPUT -p udp --dport 9000 -j ACCEPT
sudo ip6tables -A INPUT -p udp --dport 9001 -j ACCEPT

# Save rules
sudo ip6tables-save > /etc/ip6tables/rules.v6

# Verify SRT is listening on IPv6
ss -6 -ulnp | grep 9000
```

## Testing SRT over IPv6

```bash
# Test latency and connection quality
srt-live-transmit \
  "srt://[2001:db8::server]:9000?mode=caller&latency=200" \
  "null://"

# Check SRT stats (use srt-live-transmit with verbose flag)
srt-live-transmit -v \
  "srt://[2001:db8::server]:9000" \
  "file://output.ts"

# Verify IPv6 connectivity before SRT test
ping6 -c 4 2001:db8::server
nc -6 -u -w 3 2001:db8::server 9000
```

SRT's native IPv6 support through bracket-notation addressing in connection URLs (`srt://[2001:db8::server]:9000`) enables low-latency contribution links and distribution between IPv6 endpoints with the same ARQ-based reliability as IPv4 connections.
