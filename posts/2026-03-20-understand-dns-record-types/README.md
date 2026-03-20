# How to Understand DNS Record Types (A, AAAA, CNAME, MX, PTR, SRV)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, Record Types, A, CNAME, MX, PTR, SRV, Networking

Description: Understand the purpose, format, and use cases for all common DNS record types including A, AAAA, CNAME, MX, NS, SOA, PTR, TXT, and SRV records.

## Introduction

DNS stores different types of information in different record types. Each type serves a specific purpose: A records map names to IPv4 addresses, MX records route email, SRV records enable service discovery for protocols like SIP and XMPP. Understanding each type enables correct DNS configuration and faster troubleshooting.

## A and AAAA Records (Address Records)

```bash
# A: IPv4 address for a hostname

# Format: hostname. TTL IN A IPv4-address
example.com.     3600  IN  A  93.184.216.34
www.example.com. 3600  IN  A  93.184.216.34

# AAAA: IPv6 address for a hostname
example.com.     3600  IN  AAAA  2606:2800:220:1:248:1893:25c8:1946

# Query:
dig example.com A
dig example.com AAAA

# A host can have multiple A records (round-robin):
web1.example.com.  60  IN  A  10.20.0.10
web1.example.com.  60  IN  A  10.20.0.11
web1.example.com.  60  IN  A  10.20.0.12
# Resolvers return all; clients typically use the first
```

## CNAME Record (Canonical Name / Alias)

```bash
# CNAME: alias from one name to another
# Format: alias. TTL IN CNAME canonical-name.
www.example.com.    3600  IN  CNAME  example.com.
blog.example.com.   3600  IN  CNAME  mycompany.wordpress.com.
cdn.example.com.    300   IN  CNAME  mysite.cloudfront.net.

# Query:
dig www.example.com CNAME

# Important CNAME rules:
# - Cannot coexist with other records at the same name (except DNSSEC)
# - Cannot be used at the zone apex (@ or bare domain)
# - Can chain: www → www2 → canonical (but avoid deep chains)

# ANAME/ALIAS records (some providers): like CNAME but allowed at apex
# These are provider-specific, not standard DNS
```

## MX Record (Mail Exchange)

```bash
# MX: specifies mail servers for a domain
# Format: domain. TTL IN MX priority mail-server.
example.com.  3600  IN  MX  10  mail1.example.com.
example.com.  3600  IN  MX  20  mail2.example.com.   # Backup (higher = lower priority)
example.com.  3600  IN  MX  10  aspmx.l.google.com.  # Google Workspace

# Query:
dig example.com MX

# Test mail routing:
dig MX $(echo "user@example.com" | cut -d@ -f2) +short

# Priority: lower number = higher priority (10 < 20)
# Multiple MX with same priority: random selection (load balancing)
```

## NS Record (Nameserver)

```bash
# NS: authoritative nameservers for a zone
# Format: zone. TTL IN NS nameserver.
example.com.  86400  IN  NS  ns1.example.com.
example.com.  86400  IN  NS  ns2.example.com.

# Query:
dig example.com NS

# Parent zone also has NS records (delegation):
# .com zone contains:
# example.com.  172800  IN  NS  ns1.example.com.
dig @a.gtld-servers.net example.com NS   # Check parent delegation
```

## SOA Record (Start of Authority)

```bash
# SOA: authoritative information about a zone (one per zone)
dig example.com SOA +short
# Returns:
# ns1.example.com. admin.example.com. 2026032001 3600 900 604800 300
# Fields:    primary-ns    admin-email    serial  refresh retry expire neg-ttl

# The serial number must be incremented when the zone changes
# Convention: YYYYMMDDnn (e.g., 2026032001 = 2026-03-20, revision 01)
```

## PTR Record (Pointer / Reverse DNS)

```bash
# PTR: IP address to hostname (reverse DNS)
# Stored in in-addr.arpa zone
# IP reversed + .in-addr.arpa

# Format: reversed-ip.in-addr.arpa. TTL IN PTR hostname.
34.216.184.93.in-addr.arpa.  3600  IN  PTR  example.com.

# Query:
dig -x 93.184.216.34
nslookup 93.184.216.34

# Used for:
# - Email: many mail servers reject email from IPs without PTR records
# - Logging: converts IP addresses to names in logs
# - Security: verify hostname matches claimed IP
```

## TXT Record (Text)

```bash
# TXT: arbitrary text, used for verification and anti-spam
dig example.com TXT

# Common TXT record uses:
# SPF (Sender Policy Framework - email authentication):
example.com.  3600  IN  TXT  "v=spf1 include:_spf.google.com ~all"

# DMARC (email policy):
_dmarc.example.com.  3600  IN  TXT  "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"

# DKIM (email signing - per selector):
mail._domainkey.example.com.  3600  IN  TXT  "v=DKIM1; k=rsa; p=MIIBIjAN..."

# Domain ownership verification (for Google, AWS, etc.):
example.com.  TXT  "google-site-verification=abc123..."
```

## SRV Record (Service)

```bash
# SRV: service location (host, port, priority, weight)
# Format: _service._proto.name. TTL IN SRV priority weight port target.
_sip._tcp.example.com.    3600  IN  SRV  10 60 5060 sip1.example.com.
_sip._tcp.example.com.    3600  IN  SRV  10 40 5060 sip2.example.com.
_xmpp-server._tcp.example.com. 3600 IN SRV 5 0 5269 xmpp.example.com.

# Query:
dig _sip._tcp.example.com SRV

# Used by: SIP, XMPP, Kubernetes (for service discovery), some apps
# Priority: lower = preferred; Weight: for load balancing at same priority
```

## Conclusion

Each DNS record type serves a specific purpose. A/AAAA map names to IPs. CNAME creates aliases. MX routes email. NS identifies authoritative nameservers. SOA contains zone metadata. PTR provides reverse lookups. TXT stores text for email authentication and verification. SRV enables service discovery with priority and load balancing. Understanding record types is foundational to DNS configuration, troubleshooting, and security - most DNS problems reduce to missing, incorrect, or misconfigured records of a specific type.
