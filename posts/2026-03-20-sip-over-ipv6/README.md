# How to Configure SIP over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SIP, IPv6, VoIP, Telephony, SIP Proxy, Networking

Description: Configure Session Initiation Protocol (SIP) to work over IPv6 networks, covering SIP registration, invite messaging, and the important differences in SIP over IPv4 vs IPv6.

---

SIP (Session Initiation Protocol) requires specific configuration for IPv6 because SIP messages contain IP addresses in the headers (Via, Contact, SDP). IPv6 addresses in SIP must be enclosed in brackets per RFC 5118.

## SIP IPv6 Address Notation

```text
RFC 5118 "SIP" IPv6 Address Rules:
- IPv6 addresses in SIP headers MUST use brackets: [2001:db8::1]
- SIP URI format: sip:user@[2001:db8::1]:5060

Example SIP REGISTER over IPv6:
REGISTER sip:example.com SIP/2.0
Via: SIP/2.0/UDP [2001:db8::client]:5060;branch=z9hG4bK776asdhds
Max-Forwards: 70
To: sip:user@example.com
From: sip:user@example.com;tag=1928301774
Call-ID: a84b4c76e66710@2001:db8::client
CSeq: 314159 REGISTER
Contact: <sip:user@[2001:db8::client]:5060>
Content-Length: 0
```

## Testing SIP over IPv6 with sipsak

```bash
# Install sipsak

sudo apt install sipsak -y

# Send OPTIONS to SIP server over IPv6
sipsak -s sip:[2001:db8::sip-server]:5060 -v

# Test REGISTER
sipsak -s sip:user@[2001:db8::sip-server] \
  -u user \
  -a password \
  -r 3600 \
  -v

# Test with specific local IPv6 address
sipsak -s sip:[2001:db8::sip-server]:5060 \
  -b 2001:db8::client \
  -v
```

## SIP Proxy Configuration for IPv6

```bash
# OpenSIPS dual-stack configuration
# /etc/opensips/opensips.cfg

# Listen on both IPv4 and IPv6
listen=udp:0.0.0.0:5060
listen=udp:[::]:5060
listen=tcp:0.0.0.0:5060
listen=tcp:[::]:5060
listen=tls:0.0.0.0:5061
listen=tls:[::]:5061

# Route SIP over IPv6
route {
    if (is_method("REGISTER")) {
        save("location");
        exit;
    }

    # Handle IPv6 Via header
    if (af == INET6) {
        fix_nated_contact();
    }
}
```

## SIP SDP with IPv6 Addresses

```text
SDP in SIP INVITE body must specify IPv6:

v=0
o=- 123456789 123456789 IN IP6 2001:db8::caller
s=IPv6 SIP Call
c=IN IP6 2001:db8::caller
t=0 0
m=audio 10000 RTP/AVP 0 8 101
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:101 telephone-event/8000
a=fmtp:101 0-15

Key changes from IPv4:
- "IN IP6" instead of "IN IP4" in o= and c= lines
- IPv6 address NOT in brackets in SDP (unlike SIP headers)
```

## Firewall Rules for SIP over IPv6

```bash
# Allow SIP signaling over IPv6
sudo ip6tables -A INPUT -p udp --dport 5060 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 5060 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 5061 -j ACCEPT  # TLS

# Allow RTP media range
sudo ip6tables -A INPUT -p udp --dport 10000:20000 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

## Testing SIP Registration and Calls

```bash
# Test SIP REGISTER from command line
curl -X POST "http://[2001:db8::sip-api]:8080/register" \
  -d "user=1001&server=[2001:db8::sipserver]:5060"

# Use linphone-cli for testing
linphonecsh init
linphonecsh register --username 1001 \
  --password pass \
  --host [2001:db8::sipserver]

# Make test call over IPv6
linphonecsh dial sip:1002@[2001:db8::sipserver]
```

SIP over IPv6 requires careful attention to bracket notation in SIP headers (Via, Contact, Route) while SDP connection fields use IPv6 addresses without brackets, with proper NAT traversal becoming simpler since IPv6 eliminates NAT in most deployments.
