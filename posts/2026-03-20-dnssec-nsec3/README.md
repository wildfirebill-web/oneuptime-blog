# How to Configure DNSSEC NSEC3 for IPv6 Zones

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNSSEC, NSEC3, DNS, IPv6, Zone Enumeration Prevention, Security

Description: Configure DNSSEC NSEC3 authenticated denial of existence to prevent zone enumeration of IPv6 DNS zones, with correct parameter selection and operational considerations.

## NSEC vs NSEC3

| Feature | NSEC | NSEC3 |
|---|---|---|
| Zone enumeration | Yes (walks chain) | No (hashed names) |
| Performance | Slightly faster | Minimal overhead |
| Response size | Smaller | Slightly larger |
| Recommendation | Avoid for public zones | Use for public zones |

NSEC creates a linked chain of all names, enabling "zone walking" to enumerate every hostname. NSEC3 hashes domain names before linking, preventing enumeration while still providing authenticated denial.

## NSEC3 Parameters

```
NSEC3PARAM record format:
  <zone> IN NSEC3PARAM <hash-alg> <flags> <iterations> <salt>

Example:
  example.com. IN NSEC3PARAM 1 0 0 -

Parameters:
  hash-alg:   1 = SHA-1 (only option currently)
  flags:      0 = normal, 1 = Opt-Out (skip unsigned delegations)
  iterations: Number of extra SHA-1 hash iterations (0 recommended)
  salt:       Random salt (- = empty, recommended per RFC 9276)
```

## RFC 9276 Security Considerations

RFC 9276 (2022) updated NSEC3 recommendations:

```
Old recommendation: iterations=10-150, salt=random
Current recommendation (RFC 9276):
  - iterations = 0 (higher iterations don't significantly improve security)
  - salt = empty (-) (salts complicate key rollover without security benefit)

Reasoning: NSEC3 hash cracking is bounded by GPU speed, not iteration count.
Modern GPUs can crack billions of SHA-1 per second regardless of iterations.
```

## Signing a Zone with NSEC3

```bash
# Sign with NSEC3 — RFC 9276 recommended parameters
dnssec-signzone \
    -3 - \          # -3 enables NSEC3, "-" = empty salt
    -H 0 \          # 0 hash iterations
    -A \            # Sign NSEC3 records
    -N INCREMENT \  # Auto-increment serial
    -o example.com \
    -k Kexample.com.+013+KSK_ID \
    /var/named/example.com.zone \
    Kexample.com.+013+ZSK_ID

# Verify NSEC3PARAM record exists
grep "NSEC3PARAM" example.com.zone.signed
# example.com. 3600 IN NSEC3PARAM 1 0 0 -

# Check NSEC3 records exist (hashed names)
grep " NSEC3 " example.com.zone.signed | head -3
# <hash>.example.com. 3600 IN NSEC3 1 0 0 - <next_hash> AAAA A NS SOA
```

## BIND: Configure NSEC3 for Auto-Signed Zones

```
// /etc/named.conf — Configure NSEC3 for auto-signed zone

zone "example.com" {
    type master;
    file "/var/named/example.com.zone";
    key-directory "/var/named/keys/example.com";
    auto-dnssec maintain;
    inline-signing yes;

    // NSEC3 configuration
    // Set via rndc command after zone is loaded
};
```

```bash
# Set NSEC3 parameters for BIND inline-signed zone
rndc signing -nsec3param 1 0 0 - example.com

# Parameters: <hash-alg> <flags> <iterations> <salt>
# 1 = SHA-1, 0 = no opt-out, 0 = no iterations, - = empty salt

# Verify NSEC3 is active
rndc signing -status example.com | grep -i nsec

# Test: query for non-existent name — should get NSEC3 denial
dig +dnssec A nonexistent.example.com @localhost
# Should return NXDOMAIN + NSEC3 record proving it doesn't exist
```

## NSEC3 Opt-Out for Large Zones

```bash
# Opt-Out (flag=1): skip NSEC3 records for unsigned delegations
# Useful for zones with many insecure delegations (e.g., TLD zones)

# Sign with Opt-Out
dnssec-signzone \
    -3 - \
    -H 0 \
    -A \
    -A \          # -A twice enables Opt-Out behavior
    -N INCREMENT \
    -o example.com \
    -k Kexample.com.+013+KSK_ID \
    /var/named/example.com.zone \
    Kexample.com.+013+ZSK_ID

# NSEC3PARAM with Opt-Out flag:
# example.com. IN NSEC3PARAM 1 1 0 -
#                                ^ flag=1 = Opt-Out enabled
```

## Verifying NSEC3 Authenticated Denial

```bash
# Test NXDOMAIN response has NSEC3 proof
dig +dnssec +multiline AAAA doesnotexist.example.com @localhost

# Expected NSEC3-based NXDOMAIN response:
# ;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN
# ;; flags: qr aa rd; QUERY: 1, ANSWER: 0, AUTHORITY: 4, ADDITIONAL: 1
#
# ;; AUTHORITY SECTION:
# example.com. 3600 IN SOA ns1.example.com. ...
# <hash>.example.com. 3600 IN NSEC3 1 0 0 - <nexthash> A NS SOA ...
# <hash>.example.com. 3600 IN RRSIG NSEC3 ...
# <hash>.example.com. 3600 IN NSEC3 1 0 0 - <nexthash> AAAA MX ...

# The two NSEC3 records prove:
# 1. The queried name doesn't exist (no hash between two adjacent hashes)
# 2. The wildcard doesn't exist

# Attempt zone walking (should fail with NSEC3)
python3 << 'EOF'
# NSEC3 hashes are opaque — you cannot iterate through them
# to discover zone contents. This is the security benefit.
import hashlib
import base64

def nsec3_hash(name, salt=b'', iterations=0):
    """Compute NSEC3 hash for a domain name."""
    x = name.lower().encode() + salt
    for _ in range(iterations + 1):
        x = hashlib.sha1(x).digest()
    return base64.b32encode(x).decode().lower()

# Even knowing the hash function, you'd need to brute-force all possible names
print(nsec3_hash("www.example.com"))
print(nsec3_hash("doesnotexist.example.com"))
# Hashes reveal nothing about the zone contents
EOF
```

## Migrating from NSEC to NSEC3

```bash
#!/bin/bash
# migrate-nsec-to-nsec3.sh

ZONE="example.com"

# Current state: zone uses NSEC
# Target: migrate to NSEC3

# Step 1: Add NSEC3PARAM to unsigned zone file
# (BIND will pick this up with inline-signing)
rndc signing -nsec3param 1 0 0 - "${ZONE}"

# Step 2: Verify migration
sleep 5  # Allow BIND to re-sign
dig NSEC3PARAM "${ZONE}" @localhost
# Should return NSEC3PARAM record

dig NSEC3 "${ZONE}" @localhost
# Should return hashed NSEC3 records

dig +dnssec NSEC "${ZONE}" @localhost
# Should return NXDOMAIN (no NSEC records after migration)

echo "Migration complete — zone now uses NSEC3"
```

## Conclusion

NSEC3 is the recommended denial of existence mechanism for public IPv6 DNS zones because it prevents zone enumeration. Use RFC 9276 parameters: `iterations=0` and empty salt (`-`). Configure BIND inline-signed zones with `rndc signing -nsec3param 1 0 0 - zonename`. Verify with `dig +dnssec AAAA nonexistent.zone` — the NXDOMAIN response should contain NSEC3 records rather than NSEC records. Use Opt-Out (flag=1) only for zones with many unsigned delegations (like TLDs), not for leaf zones like `example.com`. NSEC3 does not provide perfect security against offline hash cracking, but eliminates trivial zone walking.
