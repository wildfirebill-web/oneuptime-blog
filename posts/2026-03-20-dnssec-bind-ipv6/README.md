# How to Configure DNSSEC with BIND for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, DNSSEC, BIND, IPv6, Security, Network

Description: A step-by-step guide to enabling DNSSEC on a BIND nameserver with full IPv6 support for signing and serving AAAA records securely.

## Overview

DNSSEC (DNS Security Extensions) adds cryptographic signatures to DNS records, protecting against cache poisoning and spoofing attacks. When combined with IPv6, you need to ensure BIND listens on IPv6 interfaces and that AAAA records are properly signed.

## Prerequisites

- BIND 9.16+ installed
- A zone file with AAAA records
- Root access on the nameserver

## Step 1: Configure BIND to Listen on IPv6

Edit `/etc/bind/named.conf.options` to listen on IPv6 interfaces:

```bash
# Open the BIND options file

sudo nano /etc/bind/named.conf.options
```

```text
options {
    // Listen on all IPv6 addresses (::) and IPv4 (0.0.0.0)
    listen-on-v6 { any; };
    listen-on { any; };

    // Allow recursive queries from your network
    allow-recursion { 192.168.0.0/16; 2001:db8::/32; };

    // Enable DNSSEC validation
    dnssec-validation auto;
};
```

## Step 2: Generate DNSSEC Keys

Generate a Key Signing Key (KSK) and a Zone Signing Key (ZSK) for your zone:

```bash
# Navigate to the zone key directory
cd /etc/bind/keys

# Generate the Zone Signing Key (ZSK) using ECDSA P-256
dnssec-keygen -a ECDSAP256SHA256 -n ZONE example.com

# Generate the Key Signing Key (KSK) with the -f KSK flag
dnssec-keygen -a ECDSAP256SHA256 -f KSK -n ZONE example.com
```

## Step 3: Configure the Zone for DNSSEC

Update your zone configuration to include the keys directory:

```text
// /etc/bind/named.conf.local
zone "example.com" {
    type master;
    file "/etc/bind/zones/db.example.com";

    // Enable inline signing - BIND signs records automatically
    dnssec-policy default;
    inline-signing yes;

    // Directory where DNSSEC keys are stored
    key-directory "/etc/bind/keys";
};
```

## Step 4: Ensure AAAA Records Exist in Your Zone File

Your zone file should include IPv6 addresses for your hosts:

```bash
; /etc/bind/zones/db.example.com
$ORIGIN example.com.
$TTL 300

@   IN  SOA  ns1.example.com. admin.example.com. (
                2026031901 ; Serial
                3600       ; Refresh
                900        ; Retry
                604800     ; Expire
                300 )      ; Minimum TTL

; Nameserver records
@   IN  NS   ns1.example.com.

; IPv4 and IPv6 addresses for the nameserver
ns1 IN  A    203.0.113.1
ns1 IN  AAAA 2001:db8:1::1

; Web server with IPv6
www IN  A    203.0.113.10
www IN  AAAA 2001:db8:1::10
```

## Step 5: Validate and Reload

```bash
# Check the configuration syntax
sudo named-checkconf

# Check the zone file syntax
sudo named-checkzone example.com /etc/bind/zones/db.example.com

# Reload BIND to apply changes
sudo rndc reload
```

## Step 6: Verify DNSSEC is Working

```bash
# Query for AAAA record with DNSSEC data (AD flag indicates authenticated data)
dig +dnssec AAAA www.example.com @2001:db8:1::1

# Check for RRSIG record (proof of signing)
dig RRSIG AAAA www.example.com @2001:db8:1::1

# Verify DS record exists in parent zone
dig DS example.com @2001:db8:1::1
```

## Monitoring with OneUptime

After configuring DNSSEC, monitor your DNS resolution over IPv6 using [OneUptime](https://oneuptime.com). Set up DNS monitors that query your AAAA records via IPv6 and alert you if DNSSEC validation fails or records become unreachable.

## Conclusion

Configuring DNSSEC with BIND for IPv6 involves listening on IPv6 interfaces, generating signing keys, and enabling inline signing. Always verify that AAAA records carry valid RRSIG signatures and publish DS records to your registrar to complete the chain of trust.
