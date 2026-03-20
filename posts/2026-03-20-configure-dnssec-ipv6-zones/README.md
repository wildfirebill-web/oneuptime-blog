# How to Configure DNSSEC for IPv6 Zones

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNSSEC, DNS Security, AAAA Records, Zone Signing

Description: A guide to signing IPv6 zones with DNSSEC to protect AAAA records from spoofing, covering key generation, zone signing, and DS record publication.

## Why DNSSEC for IPv6 Zones?

DNSSEC cryptographically signs DNS records, preventing attackers from injecting fake AAAA records and redirecting IPv6 traffic. As IPv6 adoption grows, DNSSEC becomes increasingly important for protecting AAAA records from cache poisoning.

## DNSSEC Concepts

- **ZSK (Zone Signing Key)**: Signs the zone records (AAAA, A, MX, etc.)
- **KSK (Key Signing Key)**: Signs the ZSK (used for zone trust establishment)
- **RRSIG**: Signature record attached to signed record sets
- **DNSKEY**: Public keys published in the zone
- **DS**: Delegation Signer record in the parent zone (establishes chain of trust)

## Signing a Zone with BIND dnssec-keygen

### Step 1: Generate Keys

```bash
# Create directory for DNSSEC keys
mkdir -p /etc/named/keys/example.com
cd /etc/named/keys/example.com

# Generate Zone Signing Key (ZSK) - ECDSAP256SHA256
dnssec-keygen -a ECDSAP256SHA256 -n ZONE example.com

# Generate Key Signing Key (KSK) - ECDSAP256SHA256 with -f KSK flag
dnssec-keygen -a ECDSAP256SHA256 -f KSK -n ZONE example.com

# List generated key files
ls -la
# Kexample.com.+013+12345.key    (ZSK public key)
# Kexample.com.+013+12345.private (ZSK private key)
# Kexample.com.+013+67890.key    (KSK public key)
# Kexample.com.+013+67890.private (KSK private key)
```

### Step 2: Configure BIND for Automatic DNSSEC

Modern BIND (9.16+) supports automatic DNSSEC signing:

```named
// /etc/named.conf

zone "example.com" {
    type master;
    file "/var/named/example.com.zone";

    // Enable automatic DNSSEC signing
    dnssec-policy "default";

    // Or use inline signing (simpler)
    auto-dnssec maintain;
    inline-signing yes;

    // Key directory
    key-directory "/etc/named/keys/example.com";
};
```

### Step 3: Sign the Zone Manually (if not using auto-signing)

```bash
# Sign the zone file with the ZSK and KSK
dnssec-signzone \
    -A \
    -3 $(head -c 1000 /dev/urandom | sha1sum | cut -b 1-16) \
    -N INCREMENT \
    -o example.com \
    -t \
    /var/named/example.com.zone \
    /etc/named/keys/example.com/Kexample.com.+013+12345.key \
    /etc/named/keys/example.com/Kexample.com.+013+67890.key

# This creates: example.com.zone.signed
# Update named.conf to use the signed zone file
```

### Step 4: Verify Signed AAAA Records

```bash
# Check that AAAA records are signed (RRSIG present)
dig AAAA example.com @127.0.0.1 +dnssec

# Expected output includes:
# example.com. 3600 IN AAAA 2001:db8::1
# example.com. 3600 IN RRSIG AAAA 13 2 3600 ... (signature)

# Check DNSKEY records are present
dig DNSKEY example.com @127.0.0.1

# Verify with delv (DNS lookup and validation tool)
delv @127.0.0.1 AAAA example.com
# Should show: ; fully validated
```

### Step 5: Publish DS Record to Parent Zone

The DS (Delegation Signer) record in the parent zone creates the chain of trust:

```bash
# Generate the DS record from the KSK
dnssec-dsfromkey /etc/named/keys/example.com/Kexample.com.+013+67890.key

# Output (submit this to your domain registrar):
# example.com. IN DS 67890 13 2 <hash>

# Or use dig to view DS after publication
dig DS example.com @parent-ns
```

## DNSSEC for IPv6 Reverse DNS Zones

```named
// Sign the reverse zone for IPv6
zone "8.b.d.0.1.0.0.2.ip6.arpa" {
    type master;
    file "/var/named/ip6-reverse.zone";
    dnssec-policy "default";
    inline-signing yes;
    key-directory "/etc/named/keys/ip6-reverse";
};
```

```bash
# Generate keys for the reverse zone
mkdir -p /etc/named/keys/ip6-reverse
cd /etc/named/keys/ip6-reverse
dnssec-keygen -a ECDSAP256SHA256 -n ZONE 8.b.d.0.1.0.0.2.ip6.arpa
dnssec-keygen -a ECDSAP256SHA256 -f KSK -n ZONE 8.b.d.0.1.0.0.2.ip6.arpa
```

## Verifying DNSSEC Validation

```bash
# Test that a validating resolver correctly validates the signed zone
dig AAAA www.example.com +dnssec +adflag @8.8.8.8

# The "ad" flag in the response means "authenticated data" (DNSSEC validated)
# ;; flags: qr rd ra ad;  ← "ad" flag confirms DNSSEC validation

# Check for validation failures
dig AAAA www.example.com @8.8.8.8 | grep "SERVFAIL\|NOERROR"
```

## DNSSEC Monitoring

```bash
# Check key expiration dates
for key in /etc/named/keys/example.com/*.key; do
    EXPIRE=$(dnssec-checkds -d $(dig +short DS example.com @parent) -f $key 2>/dev/null)
    echo "$key: $EXPIRE"
done

# Monitor with BIND's built-in key management
rndc dnssec -checkds example.com
```

## Summary

DNSSEC for IPv6 zones protects AAAA records from spoofing by cryptographically signing them. Generate ZSK and KSK keys with `dnssec-keygen`, configure BIND with `dnssec-policy` for automatic signing, verify with `dig AAAA <domain> +dnssec` that RRSIG records appear, and submit the DS record to your registrar to complete the chain of trust. Sign both forward and reverse IPv6 zones for complete DNSSEC coverage.
