# How to Troubleshoot DNS NXDOMAIN Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, NXDOMAIN, Troubleshooting, Linux, Debugging, Networking

Description: Diagnose DNS NXDOMAIN errors by distinguishing between non-existent domains, missing records, search domain issues, and resolver-specific problems.

## Introduction

NXDOMAIN (Non-Existent Domain) means the DNS server is definitively saying "this domain does not exist." Unlike a timeout or SERVFAIL, NXDOMAIN is a definitive negative answer from an authoritative server. Understanding the difference between a truly non-existent domain, a missing record for an existing domain, a search domain issue, or a resolver-specific NXDOMAIN is essential for diagnosis.

## Understand What NXDOMAIN Means

```bash
# NXDOMAIN = the DOMAIN NAME does not exist (not a specific record type)
# Example:
dig nonexistent.example.com
# status: NXDOMAIN → the name nonexistent.example.com doesn't exist in DNS

# NOERROR with empty ANSWER = domain exists but no record of that type
dig example.com AAAA
# status: NOERROR, ANSWER: 0 → example.com exists but has no AAAA record
# This is NOT NXDOMAIN

# The distinction matters for troubleshooting:
# NXDOMAIN: domain doesn't exist
# NOERROR + no answer: domain exists, missing record type
```

## Verify the Domain Exists

```bash
# Check if domain exists in DNS at all:
dig example.com ANY +short
# Any response (A, NS, SOA, MX...) = domain exists

# Check authoritative server directly (bypasses resolver cache):
AUTH_NS=$(dig NS example.com +short 2>/dev/null | head -1)
if [ -n "$AUTH_NS" ]; then
    dig @$AUTH_NS api.example.com
else
    echo "Could not find authoritative NS - domain may not exist"
fi

# Check SOA record (exists = domain is configured):
dig example.com SOA +short
# Returns SOA = domain exists in DNS (even if specific hostname doesn't)
```

## Diagnose Search Domain Issues

```bash
# NXDOMAIN from short hostname + search domain problem:
# /etc/resolv.conf: search company.internal
# Query: dig db → tries db.company.internal → if this doesn't exist: NXDOMAIN
# But you might have meant: db.company.local

# Check your search domains:
cat /etc/resolv.conf | grep search
# search company.internal us.company.internal

# Test without search domain (use FQDN with trailing dot):
dig db.company.internal.   # Force absolute lookup (no search)
dig db.company.local.      # Check alternative domain

# Debug search domain expansion:
strace -e trace=network getent hosts db 2>&1 | grep -i "company\|resolv"
```

## Distinguish Real vs Resolver-Specific NXDOMAIN

```bash
# Some resolvers return NXDOMAIN for blocked domains (parental control, DNS filters)
# Verify by querying multiple resolvers:

HOSTNAME="api.example.com"
for resolver in 8.8.8.8 1.1.1.1 9.9.9.9 208.67.222.222; do
    RESULT=$(dig @$resolver $HOSTNAME +short 2>/dev/null)
    CODE=$(dig @$resolver $HOSTNAME 2>/dev/null | grep "status:" | awk '{print $4}')
    echo "$resolver: code=$CODE answer=${RESULT:-empty}"
done
# If one resolver returns NXDOMAIN but others return an IP:
# → That resolver is filtering/blocking this domain (DNS-based filtering)
```

## Negative Caching of NXDOMAIN

```bash
# NXDOMAIN responses are cached for the SOA negative TTL duration:
# $TTL 3600
# @ IN SOA ns1 admin (serial refresh retry expire 300)
#                                                   ^^^
# The last value in SOA = negative TTL (300 seconds)

# Check how long NXDOMAIN was cached:
dig nonexistent.example.com
# Look at SOA in AUTHORITY section → last field = negative TTL remaining

# If NXDOMAIN is being returned for a hostname you just created:
# Wait for the negative TTL to expire, then re-query
# Or flush the resolver cache:
resolvectl flush-caches

# Force re-query bypassing resolver cache:
dig @8.8.8.8 newhost.example.com  # Google won't have your local cache
```

## NXDOMAIN Hijacking

```bash
# ISPs sometimes hijack NXDOMAIN to return their own "helpful" search page
# Symptom: NXDOMAIN query returns an IP instead of NXDOMAIN status

# Test for NXDOMAIN hijacking:
dig random-nonexistent-12345.com
# Should return: NXDOMAIN
# If returns IP: your ISP is hijacking NXDOMAIN (returns their search page IP)

# Fix: use a resolver that doesn't hijack (1.1.1.1, 8.8.8.8, 9.9.9.9):
echo "nameserver 1.1.1.1" > /etc/resolv.conf
# These resolvers pass through NXDOMAIN as-is

# Check if your current resolver hijacks:
CURRENT_NS=$(grep ^nameserver /etc/resolv.conf | head -1 | awk '{print $2}')
RESULT=$(dig @$CURRENT_NS xyz123nonexistent456.com +short)
echo "NXDOMAIN test: ${RESULT:-correctly returned empty (no hijacking)}"
```

## Conclusion

NXDOMAIN troubleshooting requires distinguishing four causes: truly missing domain (verify on authoritative server), missing specific record type (different from NXDOMAIN - check for NOERROR + empty answer), search domain expansion gone wrong (use FQDN with trailing dot to test), and resolver-specific blocking (compare multiple resolvers). Negative caching means a just-created hostname won't resolve until the SOA negative TTL expires on the querying resolver — flush the resolver cache to force immediate re-query.
