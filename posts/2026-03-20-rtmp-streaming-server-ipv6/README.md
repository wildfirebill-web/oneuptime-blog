# How to Configure RTMP Streaming Server with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RTMP, IPv6, Streaming, Nginx, Media Server, Live Streaming

Description: Configure an RTMP live streaming server to accept streams and serve viewers over IPv6, using Nginx-RTMP module with IPv6 listener and relay configuration.

---

RTMP (Real-Time Messaging Protocol) is widely used for live streaming. Configuring an RTMP server with IPv6 enables streamers and viewers on IPv6 networks to push and pull streams without IPv4 NAT traversal.

## Installing Nginx with RTMP Module

```bash
# Install Nginx with RTMP module
sudo apt install nginx libnginx-mod-rtmp -y

# Or build from source with RTMP module
sudo apt install build-essential libpcre3-dev libssl-dev zlib1g-dev -y
git clone https://github.com/arut/nginx-rtmp-module.git
wget http://nginx.org/download/nginx-1.24.0.tar.gz
tar xf nginx-1.24.0.tar.gz
cd nginx-1.24.0
./configure --add-module=../nginx-rtmp-module --with-http_ssl_module
make && sudo make install
```

## Configuring Nginx RTMP for IPv6

```nginx
# /etc/nginx/nginx.conf

worker_processes auto;

events {
    worker_connections 1024;
}

# RTMP configuration
rtmp {
    server {
        # Listen on all interfaces including IPv6
        # RTMP module listens on all interfaces by default
        listen 1935;
        listen [::]:1935;

        chunk_size 4096;

        application live {
            live on;
            record off;

            # Allow publish from any IPv6 address
            allow publish all;
            allow play all;

            # Relay to another IPv6 RTMP server
            # push rtmp://[2001:db8::relay]/live;
        }

        application vod {
            play /var/www/html/videos/;
        }
    }
}

http {
    server {
        listen 80;
        listen [::]:80;
        server_name stream.example.com;

        # HLS endpoint
        location /hls {
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /var/www/html;
            add_header Cache-Control no-cache;
        }

        # RTMP statistics
        location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }
    }
}
```

## Firewall Rules for RTMP IPv6

```bash
# Allow RTMP port over IPv6
sudo ip6tables -A INPUT -p tcp --dport 1935 -j ACCEPT

# Allow HLS/HTTP over IPv6
sudo ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Streaming to IPv6 RTMP Server

```bash
# Push stream to IPv6 RTMP server using FFmpeg
ffmpeg -i /dev/video0 \
  -c:v libx264 \
  -c:a aac \
  -f flv \
  "rtmp://[2001:db8::streamserver]/live/mystream"

# Stream from file to IPv6 RTMP
ffmpeg -re -i input.mp4 \
  -c:v libx264 \
  -c:a aac \
  -f flv \
  "rtmp://[2001:db8::streamserver]/live/test"

# OBS Studio configuration for IPv6 RTMP:
# Server: rtmp://[2001:db8::streamserver]/live
# Stream Key: mystream
```

## Playing RTMP Stream from IPv6 Server

```bash
# Pull RTMP stream from IPv6 server with FFplay
ffplay "rtmp://[2001:db8::streamserver]/live/mystream"

# Convert RTMP to HLS and serve over IPv6
ffmpeg -i "rtmp://[2001:db8::streamserver]/live/mystream" \
  -c:v copy \
  -c:a copy \
  -f hls \
  -hls_time 4 \
  -hls_list_size 5 \
  /var/www/html/hls/stream.m3u8
```

## Monitoring the RTMP Server

```bash
# Check Nginx is listening on IPv6
ss -6 -tlnp | grep 1935

# View active RTMP sessions
curl http://localhost/stat | grep "stream\|client"

# Check Nginx error log
sudo tail -f /var/log/nginx/error.log | grep -i "ipv6\|::"

# Monitor connections
sudo netstat -6 -tnp | grep 1935
```

Nginx's RTMP module supports IPv6 through the `listen [::]:1935` directive, enabling both IPv6 stream publishers (encoders like OBS/FFmpeg) and IPv6 viewers to connect to the media server using bracket-notation IPv6 RTMP URLs.
