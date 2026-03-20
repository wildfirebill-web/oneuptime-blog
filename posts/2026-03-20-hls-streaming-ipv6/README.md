# How to Configure HLS Streaming with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: HLS, IPv6, Streaming, Nginx, CDN, Live Streaming, HTTP Live Streaming

Description: Configure HTTP Live Streaming (HLS) to serve video content to viewers over IPv6, including Nginx HLS origin server setup, IPv6 listener configuration, and CDN delivery.

---

HTTP Live Streaming (HLS) is an adaptive streaming protocol that delivers video segments over HTTP. Since HLS is HTTP-based, configuring it for IPv6 primarily involves ensuring the HTTP server listens on IPv6 and generates correct IPv6-aware playlist URLs.

## Setting Up Nginx HLS Origin Server

```nginx
# /etc/nginx/nginx.conf

worker_processes auto;

events {
    worker_connections 4096;
}

rtmp {
    server {
        listen 1935;
        listen [::]:1935;

        application live {
            live on;

            # HLS output
            hls on;
            hls_path /var/www/html/hls;
            hls_fragment 4s;
            hls_playlist_length 24s;

            # Record for VOD
            # record all;
            # record_path /var/recordings;
        }
    }
}

http {
    include mime.types;
    default_type application/octet-stream;
    sendfile on;
    keepalive_timeout 65;

    server {
        listen 80;
        listen [::]:80;
        server_name hls.example.com;

        location /hls {
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /var/www/html;

            # CORS for player access from any origin
            add_header Access-Control-Allow-Origin '*';
            add_header Cache-Control no-cache;

            # Disable cache for live stream playlists
            location ~ \.m3u8$ {
                add_header Cache-Control 'no-cache, no-store, must-revalidate';
                expires -1;
            }
        }
    }
}
```

## Generating HLS with FFmpeg over IPv6

```bash
# Pull RTMP from IPv6 source and output HLS

ffmpeg -i "rtmp://[2001:db8::source]/live/stream" \
  -c:v libx264 -b:v 2000k \
  -c:a aac -b:a 128k \
  -f hls \
  -hls_time 4 \
  -hls_list_size 5 \
  -hls_segment_filename "/var/www/html/hls/seg_%03d.ts" \
  /var/www/html/hls/stream.m3u8

# Multi-bitrate HLS from IPv6 source
ffmpeg -i "rtmp://[2001:db8::source]/live/stream" \
  -filter_complex "[v:0]split=2[v1][v2]" \
  -map "[v1]" -map a:0 -c:v libx264 -b:v 3000k -c:a aac \
  -f hls -var_stream_map "v:0,a:0" \
  -master_pl_name master.m3u8 \
  /var/www/html/hls/stream_%v.m3u8
```

## Firewall for HLS over IPv6

```bash
# Allow HTTP/HTTPS for HLS delivery
sudo ip6tables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow RTMP ingress from encoders
sudo ip6tables -A INPUT -p tcp --dport 1935 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6

# Verify Nginx listening on IPv6
ss -6 -tlnp | grep -E "80|443|1935"
```

## HLS Player Configuration for IPv6

```html
<!-- HTML5 HLS Player using hls.js -->
<!DOCTYPE html>
<html>
<head>
    <title>IPv6 HLS Stream</title>
    <script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
</head>
<body>
    <video id="video" controls width="800"></video>
    <script>
        // HLS.js handles IPv6 URLs in brackets automatically
        var video = document.getElementById('video');
        var hls = new Hls();
        // For IPv6 direct access (works in most modern browsers)
        hls.loadSource('http://[2001:db8::hls-server]/hls/stream.m3u8');
        hls.attachMedia(video);
    </script>
</body>
</html>
```

## CDN Configuration for IPv6 HLS

```text
Configure CDN origin over IPv6:
- Origin server: 2001:db8::hls-origin
- CDN fetches segments from IPv6 origin
- Serves to both IPv4 and IPv6 viewers via Anycast

Cloudflare:
- Stream > Live Inputs > Create Input
- Supports IPv6 for both ingest and delivery

AWS CloudFront:
- Origin: hls.example.com (with AAAA record)
- Enables IPv6: Yes
- IPv6 viewers receive content via IPv6 PoP
```

## Testing HLS over IPv6

```bash
# Test HLS playlist is accessible over IPv6
curl -6 http://[2001:db8::hls-server]/hls/stream.m3u8

# Play HLS over IPv6 with FFplay
ffplay "http://[2001:db8::hls-server]/hls/stream.m3u8"

# Download HLS segment
curl -6 -O "http://[2001:db8::hls-server]/hls/seg_001.ts"

# Check HLS playback
vlc "http://[2001:db8::hls-server]/hls/stream.m3u8"
```

HLS over IPv6 requires only ensuring the HTTP server's `listen [::]:80` directive is active, with the RTMP-to-HLS transcoding pipeline and segment delivery working identically to IPv4 once the transport layer accepts IPv6 client connections.
