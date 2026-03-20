# How to Troubleshoot SIP over IPv6 Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SIP, IPv6, VoIP, Troubleshooting, Debugging, Connectivity

Description: Diagnose and resolve common SIP over IPv6 problems including registration failures, call setup issues, one-way audio, and SDP address mismatches.

---

SIP over IPv6 issues commonly involve incorrect IPv6 address formatting in SIP headers, firewall blocking, DNS resolution problems, or SDP/media path mismatches. Systematic diagnosis using SIP debugging tools isolates the problem layer.

## Step 1: Verify Basic IPv6 SIP Connectivity

```bash
# Test if SIP server is reachable over IPv6

nc -6 -u -w 3 2001:db8::sip-server 5060

# Send SIP OPTIONS to check server
sipsak -s sip:[2001:db8::sip-server]:5060 -v

# Expected response:
# SIP/2.0 200 OK
# or
# SIP/2.0 403 Forbidden (server reachable but not permitting)

# Test TCP SIP over IPv6
nc -6 -w 3 2001:db8::sip-server 5060 << 'EOF'
OPTIONS sip:[2001:db8::sip-server] SIP/2.0
Via: SIP/2.0/TCP [2001:db8::client]:5060;branch=z9hG4bKtest123
From: sip:test@[2001:db8::client];tag=test
To: sip:server@[2001:db8::sip-server]
Call-ID: test-call@[2001:db8::client]
CSeq: 1 OPTIONS
Content-Length: 0

EOF
```

## Step 2: Capture and Analyze SIP Messages

```bash
# Capture all SIP over IPv6
sudo tcpdump -i eth0 -nn ip6 and port 5060 -A

# Save for analysis
sudo tcpdump -i eth0 -nn ip6 and port 5060 \
  -w /tmp/sip_ipv6_debug.pcap

# Analyze with tshark
tshark -r /tmp/sip_ipv6_debug.pcap \
  -Y "sip" \
  -T fields -e sip.Request-Line -e sip.Status-Line \
  -e sip.From -e sip.To

# View full SIP messages
tshark -r /tmp/sip_ipv6_debug.pcap -Y "sip" -V | grep -A50 "REGISTER\|INVITE"
```

## Step 3: Diagnose Registration Failures

```bash
# Common registration failure reasons:

# Problem 1: Wrong Via header format
# WRONG: Via: SIP/2.0/UDP 2001:db8::client:5060
# CORRECT: Via: SIP/2.0/UDP [2001:db8::client]:5060

# Problem 2: Contact without brackets
# WRONG: Contact: <sip:user@2001:db8::client>
# CORRECT: Contact: <sip:user@[2001:db8::client]:5060>

# Check Asterisk registration errors
asterisk -rx "pjsip show registrations"
sudo tail -f /var/log/asterisk/messages | grep "REGISTER\|401\|403"

# Check Kamailio registration log
sudo tail -f /var/log/kamailio/kamailio.log | grep "REGISTER\|2001:"
```

## Step 4: Diagnose One-Way Audio

```bash
# One-way audio is often an SDP issue

# Check SDP in captured packets
tshark -r /tmp/sip_ipv6_debug.pcap \
  -Y "sdp" \
  -T fields -e sdp.connection_info \
  -e sdp.media

# SDP c= line should show IPv6:
# c=IN IP6 2001:db8::caller  (correct)
# c=IN IP4 192.168.1.1 (wrong - using IPv4 media with IPv6 signaling)

# Verify firewall allows RTP range
ss -6 -ulnp | grep -E "1[0-9]{4}|[2-5][0-9]{4}"

# Test RTP port from remote
nc -6 -u -w 3 2001:db8::client 49170 && echo "RTP port open"
```

## Step 5: NAT and Contact Header Issues

```bash
# Problem: Contact header contains private IPv6 or wrong address

# Check what Contact header is being sent
sudo tcpdump -i eth0 -nn ip6 and port 5060 -A | grep "Contact:"

# If Contact shows wrong address, configure fix in Asterisk
# /etc/asterisk/pjsip.conf:
# [transport-udp-ipv6]
# external_signaling_address=2001:db8::public-address
# external_media_address=2001:db8::public-address

# Kamailio fix_nated_contact for IPv6
# In routing script:
# if (af == INET6 && !is_present_hf("Record-Route")) {
#     fix_nated_contact();
# }
```

## Step 6: DNS Troubleshooting for SIP over IPv6

```bash
# SIP may use DNS to find server
# Check AAAA record for SIP domain
dig AAAA sip.example.com +short

# Check SRV record for SIP
dig SRV _sip._udp.example.com +short
dig SRV _sip._tcp.example.com +short
dig SRV _sips._tcp.example.com +short

# Verify DNS resolver is IPv6 capable
cat /etc/resolv.conf
ping6 $(cat /etc/resolv.conf | grep nameserver | head -1 | awk '{print $2}')
```

## Step 7: TLS/SIPS over IPv6

```bash
# Test SIPS (SIP TLS) over IPv6
openssl s_client \
  -connect [2001:db8::sip-server]:5061 \
  -tls1_2

# Verify TLS certificate covers IPv6 address
openssl s_client -connect [2001:db8::sip-server]:5061 2>/dev/null | \
  openssl x509 -noout -text | grep "Subject Alternative\|IP Address"

# Certificate should include:
# IP Address:2001:db8::sip-server
# OR DNS:sip.example.com (and server resolves to IPv6)
```

SIP over IPv6 troubleshooting centers on bracket notation in headers (Via, Contact, Route), SDP IPv6 connection lines without brackets, and verifying that both signaling and media firewalls permit the SIP port (5060) and RTP port range for IPv6 traffic.
