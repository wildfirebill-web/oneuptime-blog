# How to Configure IPv6 QoS for Video Streaming

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, QoS, Video Streaming, DSCP, Bandwidth, Linux, HLS, RTMP

Description: Configure QoS policies for video streaming traffic over IPv6 to ensure sufficient bandwidth allocation, minimize buffering, and maintain stream quality under network congestion.

---

Video streaming over IPv6 benefits from QoS policies that guarantee minimum bandwidth, manage buffer sizes, and use appropriate DSCP marking. Unlike VoIP's strict low-latency requirements, video streaming prioritizes consistent throughput with some tolerance for latency.

## Video Streaming QoS Requirements

```
Video Streaming QoS Parameters:

Live Streaming (RTMP/HLS):
- Bandwidth: Variable (720p: ~3Mbps, 1080p: ~5-8Mbps, 4K: ~15-25Mbps)
- Latency: 2-30 seconds (adaptive bitrate handles variations)
- Jitter tolerance: High (HLS buffers multiple segments)
- DSCP: AF41 (Assured Forwarding, class 4, low drop precedence)

Video Conferencing (WebRTC):
- Bandwidth: 500kbps-4Mbps per stream
- Latency: < 200ms end-to-end
- Jitter: < 100ms
- DSCP: AF41 or EF (higher priority than streaming)
```

## DSCP Marking for Video over IPv6

```bash
# /etc/nftables.conf - Video QoS for IPv6

table ip6 mangle {
    chain prerouting {
        type filter hook prerouting priority mangle; policy accept;

        # RTMP streaming - AF41
        meta l4proto tcp tcp dport 1935 ip6 dscp set af41

        # HLS (HTTP-based) - AF31
        meta l4proto tcp tcp dport { 80, 443 } ip6 dscp set af31

        # SRT streaming - AF41
        meta l4proto udp udp dport 9000-9010 ip6 dscp set af41

        # RTSP - AF41
        meta l4proto { tcp, udp } th dport 554 ip6 dscp set af41

        # RTP video streams
        meta l4proto udp udp dport 20000-30000 ip6 dscp set af41

        # Video from specific streaming server subnets
        ip6 saddr 2001:db8:streaming::/64 ip6 dscp set af41
    }
}
```

## HTB Configuration for Video Streaming

```bash
#!/bin/bash
# video_qos_ipv6.sh - QoS for video streaming over IPv6

IFACE="eth0"
TOTAL_BW="1000mbit"  # 1 Gbps interface

sudo tc qdisc del dev $IFACE root 2>/dev/null || true

# Root HTB
sudo tc qdisc add dev $IFACE root handle 1: htb default 40

# Root class
sudo tc class add dev $IFACE parent 1: classid 1:1 htb rate $TOTAL_BW

# VoIP class (highest priority)
sudo tc class add dev $IFACE parent 1:1 classid 1:10 \
  htb rate 100mbit ceil 100mbit prio 1

# Live video streaming class
sudo tc class add dev $IFACE parent 1:1 classid 1:20 \
  htb rate 500mbit ceil 800mbit prio 2

# Add FQ-CoDel to video class (manage bufferbloat)
sudo tc qdisc add dev $IFACE parent 1:20 fq_codel \
  target 20ms interval 100ms

# Interactive video (conferencing)
sudo tc class add dev $IFACE parent 1:1 classid 1:30 \
  htb rate 200mbit ceil 400mbit prio 2

# Best-effort
sudo tc class add dev $IFACE parent 1:1 classid 1:40 \
  htb rate 200mbit prio 4

# Filter: IPv6 AF41 -> video streaming class
# AF41 = DSCP 34 = 0x22, TC byte value: 34 << 2 = 0x88
sudo tc filter add dev $IFACE protocol ipv6 parent 1:0 prio 2 \
  u32 match u8 0x88 0xfc at 1 \
  flowid 1:20

echo "Video streaming IPv6 QoS configured"
```

## Nginx QoS for IPv6 Video Delivery

```nginx
# /etc/nginx/nginx.conf - Optimize for IPv6 video delivery

http {
    # Rate limiting for HLS delivery over IPv6
    limit_req_zone $binary_remote_addr zone=video_ipv6:10m rate=10r/s;

    server {
        listen [::]:443 ssl http2;

        location /hls/ {
            # Rate limit segment requests
            limit_req zone=video_ipv6 burst=20 nodelay;

            # Set buffer sizes for video delivery
            proxy_buffers 16 4m;
            proxy_buffer_size 2m;

            # HLS cache headers
            add_header Cache-Control "public, max-age=2";

            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
        }
    }
}
```

## Testing Video QoS over IPv6

```bash
# Test 1: Measure available bandwidth to IPv6 streaming server
iperf3 -6 -c 2001:db8::video-server -t 30 -P 4

# Test 2: Simulate HLS segment download over IPv6
for i in {1..5}; do
  time curl -6 -o /dev/null \
    "http://[2001:db8::server]/hls/segment_${i}.ts"
done

# Test 3: Verify DSCP marking on video packets
sudo tcpdump -i eth0 -nn ip6 \
  and "tcp port 1935" -v | grep "class 0x88\|dscp af41"

# Test 4: Continuous bandwidth monitoring
watch -n 1 'cat /proc/net/if_inet6 | awk "{print $5, $6}"'

# Test 5: Check queue drops during video streaming
sudo tc -s class show dev eth0 | grep -A5 "1:20"
```

## Multicast Video over IPv6

```bash
# IPv6 multicast for efficient video distribution

# Join multicast group (receiver)
ip -6 maddr add ff3e::100 dev eth0

# Send multicast video (sender)
ffmpeg -re -i input.mp4 \
  -c:v libx264 -b:v 3000k \
  -c:a aac -b:a 128k \
  -f mpegts \
  "udp://[ff3e::100]:1234?localaddr=2001:db8::source"

# Receive multicast
ffplay "udp://[ff3e::100]:1234"

# Firewall - allow multicast
sudo ip6tables -A INPUT -d ff3e::/16 -p udp -j ACCEPT
```

Video streaming QoS over IPv6 uses AF41 DSCP marking to provide adequate throughput guarantees without the strict latency requirements of VoIP, with HTB queuing ensuring streaming bandwidth is preserved during peak usage and FQ-CoDel preventing bufferbloat that would cause adaptive bitrate algorithms to unnecessarily reduce quality.
