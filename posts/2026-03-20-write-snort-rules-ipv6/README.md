# How to Write Snort Rules for IPv6 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Snort, IPv6, IDS Rules, Network Security, Intrusion Detection, Rule Writing

Description: Write Snort detection rules for IPv6 traffic, covering IPv6 address syntax, ICMPv6 rule options, extension header detection, and protocol-specific patterns.

---

Snort 3 supports IPv6 in its rule language with dedicated keywords for IPv6 headers, ICMPv6 types, and address matching. Understanding IPv6-specific rule options enables precise detection of IPv6-based threats.

## Snort 3 Rule Syntax for IPv6

```snort
# Basic IPv6 rule syntax
action proto src_ip src_port dir dst_ip dst_port (rule options)

# IPv6 address in rules uses standard notation
alert tcp 2001:db8::/32 any -> any 80 \
  (msg:"HTTP from documentation IPv6 range"; sid:1000; rev:1;)

# Negation of IPv6 address
alert ip6 ![::1] any -> $HOME_NET any \
  (msg:"Non-loopback IPv6 to home network"; sid:1001; rev:1;)

# Any IPv6 traffic
alert ip6 any any -> any any \
  (msg:"Any IPv6 traffic"; sid:1002; rev:1;)
```

## ICMPv6 Detection Rules

```snort
# ICMPv6 Router Advertisement (type 134)
alert icmp6 any any -> $HOME_NET any \
  (msg:"ICMPv6 Router Advertisement Received"; \
   itype:134; \
   sid:2000; rev:1;)

# ICMPv6 Neighbor Solicitation flood detection
alert icmp6 any any -> $HOME_NET any \
  (msg:"ICMPv6 Neighbor Solicitation Flood"; \
   itype:135; \
   detection_filter:track by_src,count 50,seconds 10; \
   sid:2001; rev:1;)

# ICMPv6 time exceeded (traceroute detection)
alert icmp6 any any -> $HOME_NET any \
  (msg:"ICMPv6 Time Exceeded - Traceroute"; \
   itype:3; icode:0; \
   sid:2002; rev:1;)

# ICMPv6 type 2 - packet too big (path MTU manipulation)
alert icmp6 any any -> $HOME_NET any \
  (msg:"ICMPv6 Packet Too Big - Possible MTU Attack"; \
   itype:2; \
   sid:2003; rev:1;)
```

## IPv6 Extension Header Rules

```snort
# Detect hop-by-hop options header (anomalous)
alert ip6 $EXTERNAL_NET any -> $HOME_NET any \
  (msg:"IPv6 Hop-by-Hop Extension Header"; \
   ip6_hdr:hopopts; \
   sid:3000; rev:1;)

# Detect routing header type 0 (forbidden per RFC 5095)
alert ip6 any any -> $HOME_NET any \
  (msg:"IPv6 Routing Header Type 0 Detected"; \
   ip6_hdr:routeopt; \
   sid:3001; rev:1;)

# Detect excessive extension headers (evasion technique)
alert ip6 any any -> $HOME_NET any \
  (msg:"Excessive IPv6 Extension Headers"; \
   ip6_hdr:dst,frag,hopopts,routeopt; \
   sid:3002; rev:1;)
```

## Application Layer Rules over IPv6

```snort
# HTTP exploit over IPv6
alert tcp any any -> $HTTP_SERVERS $HTTP_PORTS \
  (msg:"Web Application Attack over IPv6"; \
   flow:to_server,established; \
   content:"<script"; nocase; http_uri; \
   content:"alert("; nocase; http_uri; \
   sid:4000; rev:1;)

# SSH version scanning over IPv6
alert tcp any any -> $HOME_NET 22 \
  (msg:"SSH Version Scanning from IPv6"; \
   flow:to_server,established; \
   content:"SSH-"; depth:4; \
   detection_filter:track by_src,count 5,seconds 60; \
   sid:4001; rev:1;)

# DNS query for AAAA records (IPv6 reconnaissance)
alert udp any any -> any 53 \
  (msg:"DNS AAAA Query - IPv6 Reconnaissance"; \
   content:"|00 1C|"; offset:14; depth:2; \
   sid:4002; rev:1;)
```

## Testing Snort IPv6 Rules

```bash
# Test rule syntax
snort -c /etc/snort/snort.lua \
  --rule "alert ip6 any any -> any any (msg:\"IPv6\"; sid:99;)" \
  -T

# Test against PCAP file with IPv6 traffic
snort -c /etc/snort/snort.lua \
  -r /path/to/ipv6-traffic.pcap \
  -A alert_csv \
  --rule-path /etc/snort/rules/

# View alerts
cat /var/log/snort/alert_csv.txt | grep "IPv6"

# Capture IPv6 traffic for rule testing
tcpdump -i eth0 -w /tmp/ipv6-capture.pcap ip6 -c 1000
```

## Rule Performance Optimization

```snort
# Use fast-pattern matcher for performance
alert tcp any any -> $HOME_NET 80 \
  (msg:"Fast Pattern IPv6 HTTP"; \
   flow:established,to_server; \
   content:"attack_string"; fast_pattern; \
   sid:5000; rev:1;)

# Constrain by flow direction to reduce false positives
alert tcp any any -> $HOME_NET 443 \
  (msg:"HTTPS Anomaly over IPv6"; \
   flow:to_server,established; \
   content:"BadContent"; \
   sid:5001; rev:1;)
```

Snort 3's rule language supports IPv6 protocol keywords including `ip6_hdr` for extension header matching and `itype`/`icode` for ICMPv6 detection, providing the same detection capabilities for IPv6 traffic as the well-established IPv4 rule set.
