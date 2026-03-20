# How to Diagnose DNS Cache Poisoning Attacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, Security, Cache Poisoning, DNSSEC, Attack Detection, Linux

Description: Identify signs of DNS cache poisoning attacks, detect suspicious DNS responses, and implement DNSSEC and resolver hardening to prevent poisoning.

## Introduction

DNS cache poisoning (the Kaminsky attack and its variants) involves injecting false DNS records into a resolver's cache. Once poisoned, clients resolving a hostname get an attacker's IP instead of the legitimate one — enabling man-in-the-middle attacks, credential theft, and malware delivery. DNSSEC is the definitive protection, but detecting poisoning attempts and hardening resolvers provides defense-in-depth.

## Signs of Cache Poisoning

```bash
# 1. IP address changed unexpectedly:
# Compare what your resolver returns vs authoritative server:
DOMAIN="example.com"
RESOLVER_IP=$(dig $DOMAIN +short | head -1)
AUTH_NS=$(dig NS $DOMAIN +short | head -1)
AUTH_IP=$(dig @$AUTH_NS $DOMAIN +short | head -1)

if [ "$RESOLVER_IP" != "$AUTH_IP" ]; then
    echo "MISMATCH DETECTED!"
    echo "Your resolver: $RESOLVER_IP"
    echo "Authoritative: $AUTH_IP"
else
    echo "IPs match: $RESOLVER_IP"
fi

# 2. Domain resolves to suspicious IP:
# Check if the IP belongs to expected AS/ISP:
whois $RESOLVER_IP | grep -E "OrgName|country|netname"

# 3. TTL anomaly (poisoned record may have unusual TTL):
dig $DOMAIN | grep -A1 "ANSWER" | tail -1 | awk '{print "TTL:", $2}'
# Compare with expected TTL from authoritative server
```

## Check Multiple Resolvers

```bash
# Cache poisoning affects ONE resolver; others may return correct answer
# Querying multiple resolvers helps detect local poisoning

DOMAIN="banking.example.com"
declare -a RESOLVERS=("8.8.8.8" "1.1.1.1" "9.9.9.9" "208.67.222.222")

echo "DNS responses for $DOMAIN:"
for resolver in "${RESOLVERS[@]}"; do
    IP=$(dig @$resolver $DOMAIN +short 2>/dev/null | head -1)
    echo "  $resolver: ${IP:-timeout/empty}"
done

# If your local resolver disagrees with all public resolvers:
# Possible cache poisoning on your local resolver
```

## Monitor for Poisoning Attempts

```bash
# Enable DNS logging to detect poisoning attempts:
# In Unbound: log all DNS answers with mismatches:
cat >> /etc/unbound/unbound.conf << 'EOF'
server:
    log-queries: yes
    log-replies: yes
    log-tag-queryreply: yes
    log-local-actions: yes
EOF
unbound-control reload

# Watch for anomalies in Unbound logs:
tail -f /var/log/unbound/unbound.log | \
  awk '/BOGUS|SERVFAIL|NXDOMAIN/{print NR": "$0}' | head -50

# BIND: log queries with validation failures:
# In named.conf:
# logging {
#     channel dnssec_log { file "/var/log/named/dnssec.log"; severity dynamic; };
#     category dnssec { dnssec_log; };
# };
```

## Enable DNSSEC Validation

```bash
# DNSSEC is the definitive defense against cache poisoning:
# A poisoned response cannot pass DNSSEC signature verification

# Enable in Unbound:
cat >> /etc/unbound/unbound.conf << 'EOF'
server:
    auto-trust-anchor-file: "/var/lib/unbound/root.key"
EOF
unbound-control reload

# Test DNSSEC validation works:
dig +dnssec cloudflare.com
# Look for: flags: qr rd ra ad (AD = Authentic Data)

# Test that BOGUS domains fail:
dig bogus.dnssec-tools.org
# Should return SERVFAIL (not NOERROR) - this is correct!
# The domain is designed to test DNSSEC failure detection
```

## Resolver Hardening Against Poisoning

```bash
# 1. Use random source ports (randomized source ports make poisoning harder):
# Modern resolvers do this by default (CVE-2008-1447 fix)
# Verify BIND is using random ports:
named-checkconf && rndc status | grep "query source"

# 2. Enable 0x20 bit randomization (case randomization in queries):
# Some resolvers capitalize random letters in queries
# Responses must match case → harder to spoof
# Unbound does this by default

# 3. Check if you're behind a NAT that de-randomizes source ports:
# Some NAT devices map all DNS to the same external port
# → Significantly weakens poisoning resistance
tcpdump -i eth0 -n 'udp dst port 53' | \
  awk '{print $2}' | cut -d. -f5 | sort -u | head -20
# If only 1-2 source ports: NAT is de-randomizing (vulnerability)
```

## Verify DNS Integrity

```bash
# Regular DNS integrity check script:
#!/bin/bash
CRITICAL_DOMAINS=("banking.example.com" "auth.example.com" "payments.example.com")
EXPECTED_IPS=("10.20.0.100" "10.20.0.101" "10.20.0.102")

echo "DNS Integrity Check - $(date)"
echo "================================"

for i in "${!CRITICAL_DOMAINS[@]}"; do
    domain="${CRITICAL_DOMAINS[$i]}"
    expected="${EXPECTED_IPS[$i]}"
    current=$(dig +short "$domain" | head -1)

    if [ "$current" = "$expected" ]; then
        echo "OK: $domain → $current"
    else
        echo "ALERT: $domain expected $expected got ${current:-empty}"
        # Send alert to monitoring system here
    fi
done
```

## Conclusion

DNS cache poisoning detection requires comparing resolver responses against authoritative server responses — they should always match. Anomalies (different IPs from different resolvers, unexpected TTLs) warrant investigation. DNSSEC is the only cryptographically sound defense — enable validation on your resolver with `auto-trust-anchor-file`. Harden resolvers by ensuring source port randomization is not defeated by NAT devices. Regularly verify that critical domain resolutions return expected IP addresses using automated monitoring.
