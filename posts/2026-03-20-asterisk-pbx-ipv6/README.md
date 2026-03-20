# How to Configure Asterisk PBX with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Asterisk, IPv6, PBX, VoIP, SIP, Telephony, Linux

Description: Configure Asterisk PBX to accept SIP registrations, make and receive calls, and connect to SIP trunks over IPv6 networks.

---

Asterisk is a leading open-source PBX. Configuring it for IPv6 requires updating the SIP channel driver (PJSIP or chan_sip) to listen on IPv6 interfaces and configuring endpoints to use IPv6 addresses.

## Asterisk with PJSIP over IPv6

```ini
# /etc/asterisk/pjsip.conf

; Transport - listen on all IPv6 interfaces
[transport-udp-ipv6]
type=transport
protocol=udp
bind=::
local_net=2001:db8::/32

; Also bind to IPv4 (dual-stack)
[transport-udp-ipv4]
type=transport
protocol=udp
bind=0.0.0.0

; TLS transport over IPv6
[transport-tls-ipv6]
type=transport
protocol=tls
bind=::
cert_file=/etc/asterisk/keys/asterisk.crt
priv_key_file=/etc/asterisk/keys/asterisk.key

; IPv6 endpoint
[1001]
type=endpoint
context=from-internal
disallow=all
allow=ulaw,alaw,g729
auth=1001-auth
aors=1001
transport=transport-udp-ipv6

[1001-auth]
type=auth
auth_type=userpass
password=userpassword
username=1001

[1001-aor]
type=aor
max_contacts=5
```

## Asterisk with chan_sip (Legacy) over IPv6

```ini
# /etc/asterisk/sip.conf

[general]
bindaddr=::
bindport=5060

; Enable IPv6
ipv6=yes

; SIP domain
domain=pbx.example.com

; NAT settings (usually not needed for IPv6)
nat=no

; Transport
transport=udp,tcp

; Codec settings
disallow=all
allow=ulaw,alaw

; Register to IPv6 SIP trunk
register => user:password@[2001:db8::sip-trunk]/1001

; IPv6 SIP trunk peer
[sip-trunk-ipv6]
type=peer
host=2001:db8::sip-trunk
port=5060
fromdomain=example.com
disallow=all
allow=ulaw,alaw
insecure=invite
```

## Asterisk Dialplan for IPv6

```
# /etc/asterisk/extensions.conf

[from-internal]
; Dial internal extension
exten => _1XXX,1,Dial(PJSIP/${EXTEN})
exten => _1XXX,n,Hangup()

; Dial via IPv6 trunk
exten => _NXXNXXXXXX,1,Dial(PJSIP/${EXTEN}@sip-trunk-ipv6)
exten => _NXXNXXXXXX,n,Hangup()

[from-external]
; Receive calls from IPv6 trunk
exten => 1001,1,Answer()
exten => 1001,n,Dial(PJSIP/1001,30)
exten => 1001,n,Voicemail(1001@default)
exten => 1001,n,Hangup()
```

## RTP Configuration for IPv6

```ini
# /etc/asterisk/rtp.conf

[general]
rtpstart=10000
rtpend=20000

; RTP IPv6 binding
bindaddr=::

; DSCP marking
tos=ef      ; Expedited Forwarding for RTP
cos=5
```

## Firewall Rules for Asterisk over IPv6

```bash
# SIP over IPv6
sudo ip6tables -A INPUT -p udp --dport 5060 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 5060 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 5061 -j ACCEPT  # SIP TLS

# RTP media ports
sudo ip6tables -A INPUT -p udp --dport 10000:20000 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Testing Asterisk IPv6

```bash
# Verify Asterisk is listening on IPv6
ss -6 -ulnp | grep 5060

# Check PJSIP transport
asterisk -rx "pjsip show transports"

# Check SIP peer/endpoint status
asterisk -rx "pjsip show endpoints"
asterisk -rx "sip show peers"  # For chan_sip

# Test SIP registration from client
# Configure Linphone/MicroSIP to register to IPv6:
# Domain: [2001:db8::asterisk]:5060
# Username: 1001
# Password: userpassword

# Check registration
asterisk -rx "pjsip show contacts"

# Debug SIP messages
asterisk -rx "pjsip set logger on"
sudo tail -f /var/log/asterisk/messages | grep "INVITE\|REGISTER\|2001:"
```

Asterisk's PJSIP driver supports IPv6 through `bind=::` in the transport configuration, with the dual-transport approach (separate IPv4 and IPv6 transport objects) providing fine-grained control over which endpoints use which IP version for registration and calls.
