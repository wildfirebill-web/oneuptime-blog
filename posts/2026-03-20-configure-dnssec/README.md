# How to Configure DNSSEC for Secure DNS Lookups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, DNSSEC, Security, BIND, Linux, Cryptography

Description: Configure DNSSEC signing for a zone in BIND9 to protect DNS records from tampering and cache poisoning attacks.

## Introduction

DNSSEC (DNS Security Extensions) adds cryptographic signatures to DNS records. When a resolver receives a DNS response, it can verify the signature matches the known public key for that zone, confirming the data was not tampered with in transit. DNSSEC protects against cache poisoning attacks (where an attacker inserts false DNS records). This guide covers signing a zone with BIND9 and enabling DNSSEC validation.

## How DNSSEC Works

```
DNSSEC chain of trust:
  Root Zone (.) → signs .com TLD → signs example.com zone

Key types:
  ZSK (Zone Signing Key): signs actual DNS records (RRSIG)
  KSK (Key Signing Key): signs the ZSK; its hash (DS) goes in parent zone

Verification:
  Resolver has root zone trust anchor (pre-configured)
  Root's DNSKEY signed by root trust anchor
  .com's DS record in root zone → validates .com's KSK
  example.com's DS record in .com zone → validates example.com's KSK
  example.com's RRSIG records → validated with ZSK
  Chain is complete: answer is authenticated
```

## Enable DNSSEC Validation (Recursive Resolver)

```bash
# In /etc/bind/named.conf.options (for recursive resolver):
options {
    dnssec-validation auto;  # Uses managed-keys for root trust anchor
    # or: dnssec-validation yes;  # Requires explicit trust anchor
};

# Download root trust anchor:
unbound-anchor -a /var/lib/unbound/root.key  # For Unbound
# BIND manages its own root.key automatically with dnssec-validation auto

# Test DNSSEC validation:
dig +dnssec cloudflare.com
# Look for: AD flag (Authentic Data) = DNSSEC validated successfully

# Test DNSSEC failure detection:
dig bogus.dnssec-tools.org
# Should return: SERVFAIL (bogus domain has bad signature - tests your validator)
```

## Sign a Zone with BIND (Manual Method)

```bash
# Step 1: Generate keys for the zone:
cd /etc/bind/zones

# Generate ZSK (Zone Signing Key):
dnssec-keygen -a ECDSAP256SHA256 -n ZONE example.com
# Creates: Kexample.com.+013+12345.key and Kexample.com.+013+12345.private

# Generate KSK (Key Signing Key):
dnssec-keygen -a ECDSAP256SHA256 -f KSK -n ZONE example.com
# Creates: Kexample.com.+013+67890.key and Kexample.com.+013+67890.private

# Step 2: Include key files in zone:
cat >> /etc/bind/zones/db.example.com << 'EOF'
$INCLUDE "Kexample.com.+013+12345.key"
$INCLUDE "Kexample.com.+013+67890.key"
EOF

# Step 3: Sign the zone:
dnssec-signzone -A -3 $(head -c 16 /dev/urandom | xxd -p) \
  -N INCREMENT -o example.com \
  -t /etc/bind/zones/db.example.com
# Creates: db.example.com.signed

# Step 4: Update zone declaration to use signed file:
# In named.conf.local: file "/etc/bind/zones/db.example.com.signed";
```

## Automate Signing with BIND Inline Signing

```bash
# BIND 9.9+ supports automatic inline signing:
# /etc/bind/named.conf.local:
zone "example.com" {
    type master;
    file "/etc/bind/zones/db.example.com";  # Unsigned zone file

    # Inline signing: BIND signs automatically:
    inline-signing yes;
    auto-dnssec maintain;     # Automatically manage key signing
    key-directory "/etc/bind/keys/";

    # Or use dnssec-policy (BIND 9.16+):
    # dnssec-policy default;  # Use default DNSSEC policy
};

# Create key directory:
mkdir -p /etc/bind/keys
chown bind:bind /etc/bind/keys

# Generate keys:
cd /etc/bind/keys
dnssec-keygen -a ECDSAP256SHA256 -n ZONE example.com
dnssec-keygen -a ECDSAP256SHA256 -f KSK -n ZONE example.com
chown bind:bind /etc/bind/keys/*.{key,private}

# Reload BIND:
systemctl reload bind9
```

## Publish DS Record to Parent Zone

```bash
# After signing, you must publish the DS record to your registrar/parent zone
# The DS record creates the chain of trust

# Generate DS record:
dnssec-dsfromkey Kexample.com.+013+67890.key
# Output: example.com. IN DS 67890 13 2 <hash>

# Copy this DS record and submit to your domain registrar
# This creates the delegation from .com → example.com

# Verify DS is in parent zone:
dig @a.gtld-servers.net example.com DS
# Should show the DS record after registrar publishes it
```

## Verify DNSSEC is Working

```bash
# Comprehensive DNSSEC verification:
dig +dnssec example.com

# Check for RRSIG records (proof of signing):
dig example.com RRSIG +short

# Check for DNSKEY records:
dig example.com DNSKEY

# Verify the complete chain (using drill):
drill -TD example.com
# Shows each step of trust chain verification

# Test with DNSSEC diagnostic tools:
# https://dnssec-analyzer.verisignlabs.com/
# https://dnsviz.net/d/example.com/dnssec/
```

## Conclusion

DNSSEC requires two components: signing the zone (creating RRSIG and DNSKEY records) and publishing the DS record to the parent zone (creating the chain of trust). Use BIND's inline signing (`auto-dnssec maintain`) for automatic key management. After signing, submit the DS record to your domain registrar. Verify with `dig +dnssec` and look for the `AD` flag in recursive resolver responses. DNSSEC doesn't encrypt DNS — it only authenticates records. For privacy, combine with DNS over TLS or DNS over HTTPS.
