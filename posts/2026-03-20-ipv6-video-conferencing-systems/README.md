# How to Handle IPv6 in Video Conferencing Systems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Video Conferencing, WebRTC, SIP, Jitsi, Zoom, Enterprise

Description: Configure and troubleshoot IPv6 support in video conferencing platforms, covering WebRTC-based systems, SIP video endpoints, and enterprise conferencing infrastructure.

---

Video conferencing systems use various protocols (WebRTC, SIP, H.323) for media transport. IPv6 support varies significantly by platform and requires careful configuration of signaling, media, and TURN/STUN infrastructure.

## WebRTC-Based Video Conferencing (Jitsi Meet)

```bash
# Install Jitsi Meet with IPv6 support
sudo apt install jitsi-meet -y

# During installation, enter your FQDN: meet.example.com
# Ensure DNS has both A and AAAA records:
# meet.example.com. IN A 203.0.113.1
# meet.example.com. IN AAAA 2001:db8::meet

# Configure Nginx for IPv6
# /etc/nginx/sites-available/meet.example.com
server {
    listen 80;
    listen [::]:80;
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name meet.example.com;
    ...
}
```

```bash
# Configure Jitsi TURN server (coturn) for IPv6
# /etc/turnserver.conf
listening-ip=::
external-ip=2001:db8::meet

# Prosody XMPP for Jitsi - enable IPv6
# /etc/prosody/conf.avail/meet.example.com.cfg.lua
interfaces = { "::" }
```

## SIP Video Endpoints over IPv6

```
SIP video phones and endpoints:
- Most modern SIP endpoints (Polycom, Cisco, Yealink) support IPv6
- Configure IPv6 in endpoint settings:
  - Network > IPv6 Mode: Dual-Stack or IPv6 Only
  - Proxy/Registrar: sip:[2001:db8::sip-server]:5060

SIP REGISTER over IPv6:
REGISTER sip:example.com SIP/2.0
Via: SIP/2.0/UDP [2001:db8::client]:5060;branch=z9hG4bK
Contact: <sip:user@[2001:db8::client]:5060>
```

## Cisco Webex and Enterprise Video over IPv6

```
Cisco Webex IPv6:
- Webex uses dual-stack by default
- Media prefers IPv6 when available (Happy Eyeballs)
- No special configuration required for end users
- Enterprise with Webex Edge: configure IPv6 on WAN interface

Cisco Video Infrastructure:
- Cisco Meeting Server (CMS): enable IPv6 in System config
- Cisco Expressway: Settings > IP > IPv6 Enable
- TelePresence Server: IPv6 under Network > IPv6
```

## Zoom IPv6 Considerations

```
Zoom IPv6 Support:
- Zoom clients support IPv6 (Happy Eyeballs algorithm)
- Media may prefer IPv6 when available
- Zoom Rooms: ensure IPv6 is enabled on room system network

For on-premises Zoom Phone:
- Session Border Controller (SBC) needs IPv6 configuration
- SBC bridges IPv6 clients to IPv4 PSTN if needed

Verify Zoom IPv6 connectivity:
- During call: check network stats (Alt+Ctrl+Shift+N)
- Look for IPv6 addresses in transport info
```

## H.323 Video Conferencing over IPv6

```
H.323 over IPv6 configuration:
- Gatekeeper (OpenH323/GnuGk): enable IPv6 listener
  # GnuGk.ini
  [Gatekeeper::Main]
  EnableIPv6=1

- H.323 endpoint settings:
  - Gatekeeper address: [2001:db8::gk]:1719

# H.245 signaling in IPv6 SDP:
# c=IN IP6 2001:db8::endpoint
```

## Multipoint Control Unit (MCU) with IPv6

```bash
# Jitsi Videobridge IPv6 configuration
# /etc/jitsi/videobridge/config

# Enable IPv6
VIDEOBRIDGE_OPTIONS="--apis=rest,xmpp"

# In videobridge.conf:
# org.ice4j.ipv6.DISABLED=false
# org.ice4j.IPV6_DISABLED=false

# Restart Jitsi Videobridge
sudo systemctl restart jitsi-videobridge2
```

## Troubleshooting IPv6 Video Conferencing

```bash
# Check if STUN/TURN is providing IPv6 candidates
# Use browser console during WebRTC call:
# chrome://webrtc-internals/ (Chrome)
# about:webrtc (Firefox)

# Check ICE candidates:
# Look for "typ host" candidates with IPv6 addresses
# Look for "typ relay" from IPv6 TURN server

# Test TURN server IPv6 allocation
turnutils_uclient -6 -u user -w pass turn.example.com

# Verify media path uses IPv6
sudo tcpdump -i eth0 -nn ip6 and udp

# Check DTLS/SRTP over IPv6
sudo tcpdump -i eth0 -nn ip6 and "udp portrange 10000-60000"
```

IPv6 in video conferencing requires STUN/TURN infrastructure with IPv6 support for ICE candidate gathering, with modern WebRTC-based systems like Jitsi handling IPv6 through standard ICE mechanisms, while legacy H.323/SIP systems may require explicit IPv6 configuration on each component.
