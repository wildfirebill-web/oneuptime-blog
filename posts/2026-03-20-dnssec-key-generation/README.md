# How to Generate DNSSEC Keys for IPv6 Zones

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNSSEC, Key Generation, BIND, OpenDNSSEC, DNS Security, IPv6

Description: Generate DNSSEC Zone Signing Keys (ZSK) and Key Signing Keys (KSK) for IPv6 DNS zones using dnssec-keygen and OpenDNSSEC with modern algorithm selection.

## Key Types and Roles

| Key Type | Purpose | Size | Rotation |
|---|---|---|---|
| ZSK (Zone Signing Key) | Signs zone records (A, AAAA, MX, etc.) | 256-bit ECDSA | Every 90 days |
| KSK (Key Signing Key) | Signs the ZSK (DNSKEY record) | 256-bit ECDSA | Every 1-2 years |

The KSK creates a trust anchor chain: parent zone has DS record → DS points to KSK → KSK signs DNSKEY → DNSKEY contains ZSK → ZSK signs zone records.

## Algorithm Selection

| Algorithm | Number | Size | Recommendation |
|---|---|---|---|
| ECDSAP256SHA256 | 13 | 256-bit | Recommended (fast, small) |
| ECDSAP384SHA384 | 14 | 384-bit | Higher security |
| Ed25519 | 15 | 256-bit | Modern, very fast |
| RSASHA256 | 8 | 2048-bit | Legacy, still common |

ECDSA algorithms produce smaller keys and signatures than RSA, which reduces DNS response sizes - important for IPv6 zones with large AAAA records.

## Generating Keys with dnssec-keygen

```bash
# Create directory for keys

mkdir -p /var/named/keys/example.com
cd /var/named/keys/example.com

# Generate ZSK (Zone Signing Key)
# ECDSAP256SHA256 (algorithm 13) - recommended
dnssec-keygen \
    -a ECDSAP256SHA256 \
    -n ZONE \
    example.com

# Output files:
# Kexample.com.+013+XXXXX.key    - public key (add to zone)
# Kexample.com.+013+XXXXX.private - private key (keep secure!)

# Generate KSK (Key Signing Key)
# Add -f KSK flag
dnssec-keygen \
    -a ECDSAP256SHA256 \
    -n ZONE \
    -f KSK \
    example.com

# Output:
# Kexample.com.+013+YYYYY.key    - public KSK
# Kexample.com.+013+YYYYY.private - private KSK

# View key details
cat Kexample.com.+013+XXXXX.key
# example.com. IN DNSKEY 256 3 13 <public key data>
#              ^^^
#              256 = ZSK flag (257 would be KSK)

cat Kexample.com.+013+YYYYY.key
# example.com. IN DNSKEY 257 3 13 <public key data>
#              ^^^
#              257 = KSK flag (Zone Key + Secure Entry Point)
```

## Generating Ed25519 Keys (Modern)

```bash
# Ed25519 - algorithm 15, fastest signing
# Supported in BIND 9.12+, Knot DNS 2.7+

# ZSK with Ed25519
dnssec-keygen \
    -a ED25519 \
    -n ZONE \
    example.com

# KSK with Ed25519
dnssec-keygen \
    -a ED25519 \
    -n ZONE \
    -f KSK \
    example.com

# Ed25519 keys are very compact:
# cat Kexample.com.+015+ZZZZZ.key
# example.com. IN DNSKEY 257 3 15 <44-char base64>
```

## Generating Keys with OpenDNSSEC

```xml
<!-- OpenDNSSEC key configuration -->
<!-- /etc/opendnssec/kasp.xml -->

<KASP>
    <Policy name="default">
        <Description>Default DNSSEC policy</Description>

        <Signatures>
            <Resign>PT2H</Resign>           <!-- Re-sign every 2 hours -->
            <Refresh>P3D</Refresh>           <!-- Refresh signatures 3 days before expiry -->
            <Validity>
                <Default>P14D</Default>      <!-- 14-day signature validity -->
                <Denial>P14D</Denial>
            </Validity>
            <Jitter>PT12H</Jitter>
            <InceptionOffset>-PT3600S</InceptionOffset>
        </Signatures>

        <Denial>
            <NSEC3>
                <OptOut/>
                <Resalt>P100D</Resalt>
                <Hash>
                    <Algorithm>1</Algorithm>  <!-- SHA-1 for NSEC3 hash -->
                    <Iterations>0</Iterations> <!-- 0 iterations - RFC 9276 recommendation -->
                    <Salt length="0"/>         <!-- No salt - RFC 9276 recommendation -->
                </Hash>
            </NSEC3>
        </Denial>

        <Keys>
            <TTL>PT3600S</TTL>
            <RetireSafety>PT3600S</RetireSafety>
            <PublishSafety>PT3600S</PublishSafety>
            <Purge>P14D</Purge>

            <KSK>
                <Algorithm length="256">13</Algorithm>  <!-- ECDSAP256SHA256 -->
                <Lifetime>P365D</Lifetime>               <!-- 1 year -->
                <Repository>SoftHSM</Repository>
            </KSK>

            <ZSK>
                <Algorithm length="256">13</Algorithm>
                <Lifetime>P90D</Lifetime>                <!-- 90 days -->
                <Repository>SoftHSM</Repository>
            </ZSK>
        </Keys>

        <Zone>
            <PropagationDelay>PT3600S</PropagationDelay>
            <SOA>
                <TTL>PT3600S</TTL>
                <Minimum>PT300S</Minimum>
            </SOA>
        </Zone>

        <Parent>
            <PropagationDelay>PT9H</PropagationDelay>
            <DS>
                <TTL>PT3600S</TTL>
            </DS>
        </Parent>
    </Policy>
</KASP>
```

```bash
# OpenDNSSEC: add zone and generate keys
ods-ksmutil zone add --zone example.com --policy default
ods-ksmutil key generate --zone example.com --interval P1Y

# List generated keys
ods-ksmutil key list --zone example.com
```

## Key Security Best Practices

```bash
# Set restrictive permissions on private keys
chmod 600 Kexample.com.+013+*.private
chown named:named Kexample.com.+013+*

# Back up private keys securely
tar czf /secure-backup/example.com-keys-$(date +%Y%m%d).tar.gz \
    Kexample.com.+013+*.private

# Verify backup integrity
sha256sum /secure-backup/example.com-keys-*.tar.gz

# For production: store KSK private key in HSM (Hardware Security Module)
# Using PKCS#11 interface
dnssec-keygen -a ECDSAP256SHA256 -n ZONE -f KSK \
    -E pkcs11 \
    -s "pkcs11:token=example;object=ksk-key" \
    example.com
```

## Verifying Generated Keys

```bash
#!/bin/bash
# verify-dnssec-keys.sh

ZONE="example.com"
KEY_DIR="/var/named/keys/${ZONE}"

echo "=== DNSSEC Keys for ${ZONE} ==="
echo ""

for keyfile in ${KEY_DIR}/K${ZONE}.+*.key; do
    KEYID=$(basename "${keyfile}" .key | awk -F+ '{print $3}')
    FLAGS=$(awk '{print $4}' "${keyfile}")
    ALGO=$(awk '{print $6}' "${keyfile}")

    case "${FLAGS}" in
        256) TYPE="ZSK" ;;
        257) TYPE="KSK" ;;
        *)   TYPE="Unknown" ;;
    esac

    case "${ALGO}" in
        13) ALGO_NAME="ECDSAP256SHA256" ;;
        14) ALGO_NAME="ECDSAP384SHA384" ;;
        15) ALGO_NAME="Ed25519" ;;
        8)  ALGO_NAME="RSASHA256" ;;
        *)  ALGO_NAME="Algorithm ${ALGO}" ;;
    esac

    echo "Key ID: ${KEYID} | Type: ${TYPE} | Algorithm: ${ALGO_NAME}"
    ls -la "${KEY_DIR}/K${ZONE}.+*+${KEYID}."*
    echo ""
done
```

## Conclusion

DNSSEC key generation starts with choosing the right algorithm - ECDSAP256SHA256 (algorithm 13) is the current recommendation, offering small key sizes, fast signing, and wide support. Generate separate ZSK and KSK per zone using `dnssec-keygen`, using the `-f KSK` flag to mark key signing keys. Store private key files with strict permissions (600) and back them up immediately. For production deployments, use OpenDNSSEC for automated key lifecycle management or an HSM for KSK storage. Ed25519 (algorithm 15) is the most modern option for new deployments where all resolvers support it.
