# How to Configure FFmpeg for IPv6 Streaming

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: FFmpeg, IPv6, Streaming, Transcoding, Media, RTMP, HLS

Description: Use FFmpeg to stream, transcode, and relay media over IPv6 networks, covering input/output URLs with IPv6 addresses and network streaming options.

---

FFmpeg is the Swiss Army knife of media processing. It supports IPv6 for all network-based inputs and outputs, using bracket notation for IPv6 addresses in URLs. This enables streaming, transcoding, and relay tasks over IPv6-capable networks.

## FFmpeg IPv6 URL Syntax

```bash
# IPv6 addresses in FFmpeg URLs use bracket notation
# Protocol://[ipv6address]:port/path

# Examples:
# RTMP:   rtmp://[2001:db8::server]/live/stream
# HLS:    http://[2001:db8::server]/hls/stream.m3u8
# SRT:    srt://[2001:db8::server]:9000
# RTSP:   rtsp://[2001:db8::camera]:554/stream
# UDP:    udp://[2001:db8::receiver]:1234
# TCP:    tcp://[2001:db8::server]:9999
```

## Streaming to IPv6 RTMP Server

```bash
# Encode from file and stream to IPv6 RTMP
ffmpeg -re \
  -i input.mp4 \
  -c:v libx264 -preset veryfast -b:v 2000k \
  -c:a aac -b:a 128k \
  -f flv \
  "rtmp://[2001:db8::rtmp-server]/live/mystream"

# Stream from webcam to IPv6 RTMP
ffmpeg \
  -f v4l2 -i /dev/video0 \
  -f alsa -i default \
  -c:v libx264 -preset ultrafast \
  -c:a aac \
  -f flv \
  "rtmp://[2001:db8::rtmp-server]/live/webcam"

# Multi-bitrate output to IPv6 RTMP
ffmpeg -re -i input.mp4 \
  -filter_complex "[0:v]split=2[v1][v2]" \
  -map "[v1]" -c:v libx264 -b:v 3000k -f flv "rtmp://[2001:db8::server]/live/high" \
  -map "[v2]" -c:v libx264 -b:v 1000k -f flv "rtmp://[2001:db8::server]/live/low"
```

## Receiving Streams from IPv6 Sources

```bash
# Pull RTSP stream from IPv6 camera
ffmpeg -i "rtsp://[2001:db8::camera]:554/stream" \
  -c:v copy \
  -c:a copy \
  output.mp4

# Receive UDP multicast over IPv6
ffmpeg -i "udp://[ff3e::1]:1234" \
  -c:v copy output.ts

# Receive SRT from IPv6 sender
ffmpeg -i "srt://[2001:db8::sender]:9000?mode=caller" \
  -c:v copy output.ts

# Listen for SRT on IPv6 (listener mode)
ffmpeg -i "srt://[::]:9000?mode=listener" \
  -c:v copy output.ts
```

## HLS Generation from IPv6 Source

```bash
# Pull stream from IPv6 RTMP and create HLS
ffmpeg -i "rtmp://[2001:db8::server]/live/stream" \
  -c:v libx264 -b:v 3000k \
  -c:a aac -b:a 128k \
  -f hls \
  -hls_time 6 \
  -hls_list_size 5 \
  -hls_segment_filename "/var/www/html/hls/seg_%03d.ts" \
  /var/www/html/hls/playlist.m3u8

# Multi-bitrate HLS from IPv6 source
ffmpeg -i "rtmp://[2001:db8::server]/live/stream" \
  -filter_complex "[0:v]split=3[v1][v2][v3]" \
  -map "[v1]" -c:v libx264 -b:v 4000k -s 1920x1080 \
  -map "[v2]" -c:v libx264 -b:v 2000k -s 1280x720 \
  -map "[v3]" -c:v libx264 -b:v 800k -s 854x480 \
  -map a:0 -map a:0 -map a:0 \
  -f hls -var_stream_map "v:0,a:0 v:1,a:1 v:2,a:2" \
  -master_pl_name master.m3u8 \
  /var/www/html/hls/stream_%v.m3u8
```

## FFmpeg Network Relay over IPv6

```bash
# Relay RTMP from IPv4 to IPv6 destination
ffmpeg -i "rtmp://source-server/live/stream" \
  -c:v copy -c:a copy \
  -f flv \
  "rtmp://[2001:db8::ipv6-server]/live/stream"

# Relay UDP multicast to IPv6 unicast
ffmpeg -i "udp://239.0.0.1:1234" \
  -c:v copy \
  -f mpegts \
  "udp://[2001:db8::receiver]:1234"
```

## FFmpeg IPv6 Network Options

```bash
# Force IPv6 connection (even if hostname has A record)
ffmpeg -i "rtmp://stream.example.com/live/key" \
  -protocol_whitelist rtmp,tcp \
  -rw_timeout 5000000 \
  -c:v copy output.ts

# Specify IPv6 source interface
ffmpeg -i "rtmp://[2001:db8::server]/live/stream" \
  -c:v copy \
  output.ts

# Check FFmpeg network support
ffmpeg -protocols | grep -E "rtmp|srt|udp|tcp|http"
```

FFmpeg's native support for IPv6 URL notation using brackets (`[2001:db8::address]`) enables all streaming and transcoding workflows to function identically over IPv6 as over IPv4, with the URL syntax being the only change required.
