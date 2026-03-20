# How to Configure Icecast with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Icecast, IPv6, Audio Streaming, Internet Radio, Linux, Self-Hosted

Description: Configure Icecast internet radio streaming server to accept source connections and serve listeners over IPv6, enabling audio streaming on IPv6-capable networks.

---

Icecast is a streaming media server for audio and video. It supports IPv6 through its hostname binding configuration, enabling internet radio stations to serve both IPv4 and IPv6 listeners simultaneously.

## Installing Icecast

```bash
# Ubuntu/Debian

sudo apt install icecast2 -y

# Configuration wizard runs during install
# Set passwords carefully

# RHEL/CentOS
sudo dnf install icecast -y

# Check version
icecast -v
```

## Configuring Icecast for IPv6

```xml
<!-- /etc/icecast2/icecast.xml -->

<icecast>
    <location>Earth</location>
    <admin>admin@example.com</admin>

    <limits>
        <clients>100</clients>
        <sources>2</sources>
        <queue-size>524288</queue-size>
        <client-timeout>30</client-timeout>
        <header-timeout>15</header-timeout>
        <source-timeout>10</source-timeout>
    </limits>

    <authentication>
        <source-password>sourcepassword</source-password>
        <relay-password>relaypassword</relay-password>
        <admin-user>admin</admin-user>
        <admin-password>adminpassword</admin-password>
    </authentication>

    <hostname>stream.example.com</hostname>

    <!-- Listen on all interfaces including IPv6 -->
    <listen-socket>
        <port>8000</port>
        <!-- Bind to all IPv4 and IPv6 interfaces -->
        <bind-address>::</bind-address>
    </listen-socket>

    <!-- Or bind to specific IPv6 address -->
    <!--
    <listen-socket>
        <port>8000</port>
        <bind-address>2001:db8::1</bind-address>
    </listen-socket>
    -->

    <!-- Also listen on IPv4 if needed -->
    <listen-socket>
        <port>8000</port>
        <bind-address>0.0.0.0</bind-address>
    </listen-socket>

    <paths>
        <logdir>/var/log/icecast2</logdir>
        <webroot>/usr/share/icecast2/web</webroot>
        <adminroot>/usr/share/icecast2/admin</adminroot>
    </paths>

    <logging>
        <accesslog>access.log</accesslog>
        <errorlog>error.log</errorlog>
        <loglevel>3</loglevel>
    </logging>

    <security>
        <chroot>0</chroot>
    </security>
</icecast>
```

## Starting Icecast with IPv6

```bash
# Start Icecast
sudo systemctl start icecast2
sudo systemctl enable icecast2

# Verify listening on IPv6
ss -6 -tlnp | grep 8000

# Check logs
sudo tail -f /var/log/icecast2/error.log
```

## Configuring Source (Liquidsoap over IPv6)

```ruby
# /etc/liquidsoap/radio.liq - Liquidsoap source script

# Define audio source
source = playlist("/var/music/", mode="normal")

# Output to Icecast over IPv6
output.icecast(%mp3,
    host="2001:db8::icecast-server",
    port=8000,
    password="sourcepassword",
    mount="/stream.mp3",
    name="My IPv6 Radio",
    description="Streaming over IPv6",
    source)
```

## Firewall Rules for Icecast IPv6

```bash
# Allow Icecast listener port
sudo ip6tables -A INPUT -p tcp --dport 8000 -j ACCEPT

# Allow SSL Icecast (if using)
sudo ip6tables -A INPUT -p tcp --dport 8443 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Testing Icecast over IPv6

```bash
# Test Icecast web interface over IPv6
curl -6 http://[2001:db8::icecast]:8000/

# Test stream playback over IPv6
ffplay "http://[2001:db8::icecast]:8000/stream.mp3"

# VLC playback
vlc "http://[2001:db8::icecast]:8000/stream.mp3"

# Check listener statistics
curl -6 -u admin:adminpassword \
  "http://[2001:db8::icecast]:8000/admin/stats"
```

## Icecast Relay over IPv6

```xml
<!-- Relay configuration in icecast.xml -->
<relay>
    <!-- Relay from IPv6 master server -->
    <server>2001:db8::master-icecast</server>
    <port>8000</port>
    <mount>/source.mp3</mount>
    <local-mount>/relay.mp3</local-mount>
    <relay-shoutcast-metadata>1</relay-shoutcast-metadata>
</relay>
```

Icecast's `bind-address` using `::` enables dual-stack listening where both IPv4 and IPv6 listeners can connect to the same stream, making it straightforward to serve internet radio to the growing number of IPv6-capable network connections.
