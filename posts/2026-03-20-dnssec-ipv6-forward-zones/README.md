# How to Sign IPv6 Forward DNS Zones with DNSSEC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNSSEC, IPv6, DNS, BIND, Zone Signing, Security

Description: Sign IPv6 forward DNS zones with DNSSEC using BIND 9, including key generation, zone signing, and maintaining signed zones with automatic re-signing.

## DNSSEC for IPv6 Forward Zones

DNSSEC signing of forward zones that contain AAAA records is identical to signing zones with A records — the signing process is record-type agnostic. The zone file contains AAAA records alongside A records, and DNSSEC signs all records including AAAA, CNAME, MX, and others.

Benefits for IPv6:
- Validates that AAAA records are authentic
- Prevents DNS hijacking that redirects IPv6 traffic
- Provides authenticated denial of existence (NSEC/NSEC3)

## Zone File with AAAA Records

```
; /var/named/example.com.zone
$ORIGIN example.com.
$TTL 3600

@   IN SOA ns1.example.com. admin.example.com. (
            2026032001  ; serial
            3600        ; refresh
            900         ; retry
            604800      ; expire
            300 )       ; minimum

    IN NS  ns1.example.com.
    IN NS  ns2.example.com.

; IPv4 and IPv6 for name servers
ns1 IN A    203.0.113.1
ns1 IN AAAA 2001:db8::1

ns2 IN A    203.0.113.2
ns2 IN AAAA 2001:db8::2

; Dual-stack web server
www IN A    203.0.113.10
www IN AAAA 2001:db8::10

; IPv6-only service
api IN AAAA 2001:db8::20

; AAAA for multiple addresses (load balancing)
cdn IN AAAA 2001:db8:cdn::1
cdn IN AAAA 2001:db8:cdn::2
cdn IN AAAA 2001:db8:cdn::3
```

## Step 1: Generate DNSSEC Keys

```bash
# Navigate to zone key directory
cd /var/named/keys/

# Generate Zone Signing Key (ZSK) — smaller, rotated frequently
dnssec-keygen -a ECDSAP256SHA256 \
              -n ZONE \
              example.com

# Output: Kexample.com.+013+XXXXX.key and .private

# Generate Key Signing Key (KSK) — larger, updated rarely
dnssec-keygen -a ECDSAP256SHA256 \
              -n ZONE \
              -f KSK \
              example.com

# Output: Kexample.com.+013+YYYYY.key and .private

# List generated keys
ls -la Kexample.com.*
```

## Step 2: Sign the Zone

```bash
# Sign the zone with both ZSK and KSK
# NSEC3 for authenticated denial (recommended)
dnssec-signzone \
    -a \
    -3 $(head -c 4 /dev/urandom | xxd -p) \
    -A \
    -N INCREMENT \
    -o example.com \
    -k /var/named/keys/Kexample.com.+013+YYYYY \
    /var/named/example.com.zone \
    /var/named/keys/Kexample.com.+013+XXXXX

# Output: example.com.zone.signed

# Verify signed zone
dnssec-verify -o example.com example.com.zone.signed

# Check AAAA record signatures are present
grep -A2 "AAAA" example.com.zone.signed | grep RRSIG | head -5
```

## Step 3: BIND Configuration

```
// /etc/named.conf — BIND zone configuration

zone "example.com" {
    type master;
    file "/var/named/example.com.zone.signed";

    // Allow DNSSEC key rollover notifications
    key-directory "/var/named/keys";

    // Automatic signing (BIND 9.9+)
    auto-dnssec maintain;
    inline-signing yes;
};

// Enable DNSSEC validation globally
options {
    dnssec-validation auto;  // Use IANA trust anchors
    dnssec-enable yes;
};
```

## Step 4: BIND Automatic Signing (BIND 9.9+)

```bash
# With inline-signing, BIND signs zones automatically
# No manual dnssec-signzone needed

# Just update the unsigned zone file
# BIND creates .signed automatically

# Reload the zone
rndc reload example.com

# Check signing status
rndc signing -status example.com
# Output:
# Signing with key: ... (ZSK)
# Signing with key: ... (KSK) - key signing key

# Verify AAAA records have signatures
dig +dnssec AAAA www.example.com @localhost | grep -E "AAAA|RRSIG"
```

## Step 5: Verify DNSSEC Signatures

```bash
# Verify AAAA record signature
dig +dnssec +multiline AAAA www.example.com @localhost

# Expected output includes RRSIG record:
# www.example.com. 3600 IN AAAA 2001:db8::10
# www.example.com. 3600 IN RRSIG AAAA 13 3 3600 (
#     20260420000000 20260320000000 XXXXX example.com.
#     <signature data> )

# Full DNSSEC chain validation
dig +dnssec +cd AAAA www.example.com
# +cd = checking disabled (shows raw answer without local validation)

# Validate via external resolver
dig @8.8.8.8 +dnssec AAAA www.example.com | grep -E "AAAA|RRSIG|ad"
# "ad" flag in flags = Authenticated Data (DNSSEC validated)
```

## Automation: Zone Signing Script

```bash
#!/bin/bash
# sign-zone.sh — Re-sign zone and reload BIND

ZONE="example.com"
ZONE_FILE="/var/named/${ZONE}.zone"
KEY_DIR="/var/named/keys"
ZSK=$(ls ${KEY_DIR}/K${ZONE}.+013+*.key | grep -v KSK | head -1 | sed 's/\.key//')
KSK=$(ls ${KEY_DIR}/K${ZONE}.+013+*.key | xargs grep -l "key signing" | head -1 | sed 's/\.key//')

# Increment serial
SERIAL=$(grep "serial" "${ZONE_FILE}" | awk '{print $1}')
NEW_SERIAL=$(date +%Y%m%d)01
sed -i "s/${SERIAL}/${NEW_SERIAL}/" "${ZONE_FILE}"

# Sign with NSEC3
dnssec-signzone \
    -3 $(openssl rand -hex 4) \
    -A -N INCREMENT \
    -o "${ZONE}" \
    -k "${KSK}" \
    "${ZONE_FILE}" \
    "${ZSK}"

# Verify
dnssec-verify -o "${ZONE}" "${ZONE_FILE}.signed"

# Reload
rndc reload "${ZONE}"
echo "Zone ${ZONE} signed and reloaded (serial: ${NEW_SERIAL})"
```

## Conclusion

DNSSEC signing of IPv6 forward zones is straightforward — AAAA records are signed exactly like A records. Use ECDSAP256SHA256 algorithm for modern key generation (smaller, faster than RSA). Enable `inline-signing yes` in BIND to automate re-signing after zone changes. Always use NSEC3 (`-3` flag in `dnssec-signzone`) to prevent zone enumeration — without it, NSEC records reveal all names in the zone. After signing, verify AAAA records have RRSIG records with `dig +dnssec`, and confirm the `ad` (Authenticated Data) flag appears in validating resolver responses.
