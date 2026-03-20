# How to Verify DNS Propagation After Record Changes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, Propagation, TTL, Verification, Networking, Linux

Description: Verify that DNS record changes have propagated to resolvers worldwide by querying authoritative servers directly and checking multiple public resolvers from different regions.

## Introduction

After making a DNS change, the new records must propagate through the global DNS cache hierarchy. "Propagation" actually means waiting for old cached entries to expire (governed by TTL) and for recursive resolvers worldwide to re-query authoritative servers. This can take minutes (for short TTLs) to 48 hours (for long TTLs). This guide covers techniques to verify propagation status.

## Check Authoritative Servers First

```bash
# The authoritative server has the new record immediately after the change.

# If auth server shows new record but resolvers don't: still propagating (waiting for TTL)

# Find authoritative nameservers:
dig NS example.com +short
# Returns: ns1.example.com. ns2.example.com.

# Query each authoritative server directly:
for ns in $(dig NS example.com +short); do
    echo -n "$ns: "
    dig @$ns example.com A +short 2>/dev/null || echo "timeout"
done

# Auth server shows new IP = change is correct; just waiting for TTL to expire
# Auth server shows old IP = DNS record not saved/deployed yet
```

## Check Multiple Public Resolvers

```bash
# Query resolvers from different geographic locations:
# (These may have different cache states depending on when they last queried)
declare -A RESOLVERS=(
    ["Google"]="8.8.8.8"
    ["Cloudflare"]="1.1.1.1"
    ["OpenDNS"]="208.67.222.222"
    ["Quad9"]="9.9.9.9"
    ["Comodo"]="8.26.56.26"
)

DOMAIN="example.com"
echo "Checking propagation for $DOMAIN:"
for name in "${!RESOLVERS[@]}"; do
    IP=$(dig @${RESOLVERS[$name]} $DOMAIN +short 2>/dev/null)
    echo "  $name (${RESOLVERS[$name]}): ${IP:-no response}"
done
```

## Check Current TTL

```bash
# The TTL tells you how much longer old records will be cached:
dig example.com | grep -A2 "ANSWER SECTION"
# Shows: example.com. 3600 IN A 93.184.216.34
#                     ^^^^ TTL in seconds remaining

# If TTL was originally 86400 (24 hours):
# And change was made 1 hour ago:
# Remaining cache time = 23 hours at resolvers that cached it

# Track TTL countdown over time:
for i in $(seq 1 5); do
    TTL=$(dig example.com | grep -A1 "ANSWER SECTION" | tail -1 | awk '{print $2}')
    echo "$(date +%H:%M:%S): TTL=$TTL"
    sleep 60
done
# TTL should decrease by approximately 60 each minute
```

## Use Online Propagation Checkers

```bash
# These tools query from multiple global locations:
# - https://dnschecker.org
# - https://www.whatsmydns.net
# - https://dnspropagation.net
# - https://mxtoolbox.com/SuperTool.aspx

# Command-line equivalent: use DNS queries from different VPS/cloud regions:
# AWS (US East):
ssh user@ec2-us-east-1 "dig +short example.com"
# AWS (EU West):
ssh user@ec2-eu-west-1 "dig +short example.com"
# If you have servers in multiple regions
```

## Minimize Propagation Time

```bash
# Best practice: reduce TTL BEFORE making the change
# Standard approach:
# Step 1 (24-48 hours before change): reduce TTL to 300 (5 minutes)
# Verify TTL is reduced:
dig example.com | grep -A1 "ANSWER SECTION" | grep "300"

# Step 2 (make the change): update record to new value
# Step 3 (after propagation is confirmed): restore TTL to 3600+

# Check if TTL is already low enough for quick propagation:
TTL=$(dig example.com | grep -A1 "ANSWER SECTION" | tail -1 | awk '{print $2}')
echo "Current TTL: $TTL seconds ($(echo "$TTL/60" | bc) minutes)"
if [ $TTL -lt 300 ]; then
    echo "TTL is low - changes will propagate quickly"
else
    echo "TTL is high - changes may take up to $TTL seconds to propagate"
fi
```

## Verify Specific Record Types

```bash
# A record propagation:
dig example.com A +short

# CNAME change:
dig www.example.com CNAME +short

# MX record change (email routing):
dig example.com MX +short
# Test if mail delivery works after MX change:
# swaks --to test@example.com --server $(dig example.com MX +short | awk '{print $2}')

# TXT record (SPF, DMARC, domain verification):
dig example.com TXT +short

# NS record change (nameserver delegation):
# NS changes propagate through parent zone (takes longer)
dig example.com NS +short
dig @a.gtld-servers.net example.com NS  # Check from .com TLD servers directly
```

## Conclusion

DNS propagation verification requires checking the authoritative server first (should show new record immediately), then multiple public resolvers (show propagation status). The TTL of the old record determines maximum propagation time - a 3600-second TTL means up to 1 hour for all caches to expire. The best practice is reducing TTL to 300 seconds 24 hours before any planned DNS change, making the change, verifying it on authoritative servers, then confirming propagation to public resolvers within 5 minutes.
