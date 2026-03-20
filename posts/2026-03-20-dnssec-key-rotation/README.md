# How to Automate DNSSEC Key Rotation for IPv6 Zones

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNSSEC, Key Rotation, ZSK, KSK, Automation, BIND, DNS Security

Description: Automate DNSSEC Zone Signing Key (ZSK) and Key Signing Key (KSK) rotation with pre-publication, DS transitions, and monitoring to maintain continuous DNSSEC validation.

## Key Rotation Overview

| Key | Frequency | Method | Critical Step |
|---|---|---|---|
| ZSK | Every 90 days | Pre-publication | Double-sign during transition |
| KSK | Every 1-2 years | Double-KSK | Update DS at parent |

Key rotation must be done carefully - a mistake breaks DNSSEC validation for your entire zone, causing SERVFAIL for all queries.

## ZSK Rotation: Pre-Publication Method

```text
Timeline:
Day 0:   Generate new ZSK
Day 0-7: Publish new ZSK alongside old ZSK (DNS TTL propagation)
Day 7:   Start signing with new ZSK (old still in zone)
Day 14:  Remove old ZSK from zone
```

```bash
#!/bin/bash
# zsk-rotation.sh - Automated ZSK pre-publication rotation

ZONE="example.com"
KEY_DIR="/var/named/keys/${ZONE}"
ZONE_FILE="/var/named/${ZONE}.zone"

# Phase 1: Generate new ZSK

new_zsk=$(dnssec-keygen -a ECDSAP256SHA256 -n ZONE "${ZONE}")
echo "Phase 1: New ZSK generated: ${new_zsk}"

# Find old ZSK (current active ZSK)
old_zsk=$(ls ${KEY_DIR}/K${ZONE}.+013+*.key | \
    xargs -I {} bash -c 'grep -l "256 3" {}' | \
    grep -v "$(basename ${new_zsk})" | \
    head -1 | sed 's/\.key//')

echo "Old ZSK: ${old_zsk}"
echo "New ZSK: ${new_zsk}"
echo ""

# Phase 2: Sign with BOTH ZSKs (double-signing)
# The zone file contains both DNSKEY records
dnssec-signzone \
    -3 - -H 0 -A -N INCREMENT \
    -o "${ZONE}" \
    -k "${KEY_DIR}/K${ZONE}.+013+KSK" \
    "${ZONE_FILE}" \
    "${old_zsk}" \
    "${new_zsk}"   # Both ZSKs sign the zone

rndc reload "${ZONE}"

echo "Phase 2: Zone now signed with both ZSKs"
echo "WAIT: At least ${TTL} seconds before Phase 3"
echo "ACTION: Re-run with --phase3 after TTL expiry"
```

## ZSK Rotation: Phase 3 (Remove Old ZSK)

```bash
#!/bin/bash
# zsk-rotation-phase3.sh - Remove old ZSK

ZONE="example.com"
KEY_DIR="/var/named/keys/${ZONE}"
ZONE_FILE="/var/named/${ZONE}.zone"

NEW_ZSK="Kexample.com.+013+NEWZSK_ID"
OLD_ZSK="Kexample.com.+013+OLDZSK_ID"
KSK="Kexample.com.+013+KSK_ID"

# Sign with ONLY the new ZSK
dnssec-signzone \
    -3 - -H 0 -A -N INCREMENT \
    -o "${ZONE}" \
    -k "${KEY_DIR}/${KSK}" \
    "${ZONE_FILE}" \
    "${KEY_DIR}/${NEW_ZSK}"

rndc reload "${ZONE}"

# Archive old ZSK (don't delete immediately)
mkdir -p "${KEY_DIR}/archive"
mv "${KEY_DIR}/${OLD_ZSK}.key" "${KEY_DIR}/archive/"
mv "${KEY_DIR}/${OLD_ZSK}.private" "${KEY_DIR}/archive/"

echo "ZSK rotation complete"
echo "Old ZSK archived to ${KEY_DIR}/archive/"

# Verify
dig +dnssec AAAA www.${ZONE} @localhost | grep -E "RRSIG|ad"
```

## KSK Rotation: Double-KSK Method

```bash
#!/bin/bash
# ksk-rotation.sh - KSK rotation (requires DS update at registrar)

ZONE="example.com"
KEY_DIR="/var/named/keys/${ZONE}"
ZONE_FILE="/var/named/${ZONE}.zone"

# Phase 1: Generate new KSK
echo "=== Phase 1: Generate new KSK ==="
new_ksk=$(dnssec-keygen -a ECDSAP256SHA256 -n ZONE -f KSK "${ZONE}")
echo "New KSK: ${new_ksk}"
echo ""

# Generate DS record for new KSK
echo "New DS record (submit to registrar):"
dnssec-dsfromkey -2 "${KEY_DIR}/${new_ksk}.key"
echo ""

# Find old KSK
old_ksk=$(ls ${KEY_DIR}/K${ZONE}.+013+*.key | \
    xargs -I {} bash -c 'grep -l "257 3" {}' | \
    grep -v "${new_ksk}" | head -1 | sed 's/\.key//')

# Find active ZSK
zsk=$(ls ${KEY_DIR}/K${ZONE}.+013+*.key | \
    xargs -I {} bash -c 'grep -l "256 3" {}' | head -1 | sed 's/\.key//')

echo "=== Phase 2: Sign with BOTH KSKs ==="
# Zone now contains DNSKEY records for both KSKs
dnssec-signzone \
    -3 - -H 0 -A -N INCREMENT \
    -o "${ZONE}" \
    -k "${KEY_DIR}/${old_ksk}" \
    -k "${KEY_DIR}/${new_ksk}" \
    "${ZONE_FILE}" \
    "${KEY_DIR}/${zsk}"

rndc reload "${ZONE}"
echo "Both KSKs in zone - submit new DS record to registrar NOW"
echo "WAIT: Until old DS TTL expires (check dig DS ${ZONE} @parent)"
```

```bash
# Phase 3: Complete KSK rotation (after DS transition)
#!/bin/bash
# ksk-rotation-phase3.sh

ZONE="example.com"
KEY_DIR="/var/named/keys/${ZONE}"

NEW_KSK="Kexample.com.+013+NEW_KSK_ID"
OLD_KSK="Kexample.com.+013+OLD_KSK_ID"
ZSK="Kexample.com.+013+ZSK_ID"

# Verify new DS is published at parent
echo "Checking DS publication..."
dig +short DS "${ZONE}" @a.gtld-servers.net

# Sign with ONLY new KSK
dnssec-signzone \
    -3 - -H 0 -A -N INCREMENT \
    -o "${ZONE}" \
    -k "${KEY_DIR}/${NEW_KSK}" \
    "/var/named/${ZONE}.zone" \
    "${KEY_DIR}/${ZSK}"

rndc reload "${ZONE}"

# Verify validation still works
dig +dnssec AAAA www.${ZONE} | grep -c "ad" && echo "PASS: Validation working"

# Archive old KSK
mkdir -p "${KEY_DIR}/archive"
mv "${KEY_DIR}/${OLD_KSK}"* "${KEY_DIR}/archive/"
echo "KSK rotation complete"
```

## BIND: Automated Key Rollover

```text
// BIND 9.16+ supports fully automated key management
// /etc/named.conf

dnssec-policy "standard" {
    keys {
        ksk key-directory lifetime 1y algorithm ecdsap256sha256;
        zsk key-directory lifetime 90d algorithm ecdsap256sha256;
    };
    // NSEC3 configuration
    nsec3param iterations 0 optout no salt-length 0;

    // Signature validity
    signatures-validity 14d;
    signatures-refresh 5d;
    signatures-jitter 12h;
};

zone "example.com" {
    type master;
    file "/var/named/example.com.zone";
    dnssec-policy "standard";
    inline-signing yes;
};
```

```bash
# BIND manages keys automatically:
# - Generates new ZSK/KSK when needed
# - Pre-publishes new keys
# - Double-signs during transition
# - Retires old keys
# - Alerts for KSK rollover (requires manual DS update)

# Check key rollover status
rndc dnssec -status example.com

# For KSK rollover - BIND alerts:
# "zone example.com: key 12345/ECDSAP256SHA256 is ready for ksk-roll"
# Then submit new DS to registrar and confirm:
rndc dnssec -rollover -key 12345 example.com
```

## Monitoring Key Rollover Schedule

```bash
#!/bin/bash
# monitor-key-schedule.sh - Show upcoming key rotations

ZONES=("example.com" "ip6.arpa.reverse")

for ZONE in "${ZONES[@]}"; do
    echo "=== ${ZONE} ==="
    KEY_DIR="/var/named/keys/${ZONE}"

    for keyfile in ${KEY_DIR}/K${ZONE}.+*.key; do
        KEY_ID=$(basename "${keyfile}" .key | awk -F+ '{print $3}')
        FLAGS=$(awk '{print $4}' "${keyfile}")
        case "${FLAGS}" in
            256) TYPE="ZSK" ;;
            257) TYPE="KSK" ;;
        esac

        # Check private key creation time
        CREATED=$(stat -c %y "${KEY_DIR}/K${ZONE}.+013+${KEY_ID}.private" 2>/dev/null | cut -d' ' -f1)
        echo "  ${TYPE} ${KEY_ID}: created ${CREATED}"
    done
    echo ""
done
```

## Conclusion

DNSSEC key rotation follows the pre-publication model: generate the new key, publish it alongside the old key, wait for DNS TTL propagation, start signing with the new key, then remove the old key. ZSK rotation is fully automatic and takes days. KSK rotation requires a manual step: updating the DS record at the registrar, which takes hours to propagate. BIND 9.16+ `dnssec-policy` automates ZSK rotation entirely and alerts for KSK rollover (which still requires registrar interaction). Monitor upcoming key expirations with a cron job, and always verify `dig +dnssec` with the `ad` flag after each rotation step.
