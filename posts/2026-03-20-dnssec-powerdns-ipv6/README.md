# How to Configure DNSSEC with PowerDNS for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, DNSSEC, PowerDNS, IPv6, Security

Description: Learn how to enable DNSSEC on a PowerDNS authoritative nameserver with IPv6 support for signing and serving AAAA records.

## Overview

PowerDNS Authoritative Server offers built-in DNSSEC support with easy key management via the `pdnsutil` command-line tool. This guide covers enabling IPv6 listening and signing a zone that includes AAAA records.

## Prerequisites

- PowerDNS Authoritative Server 4.5+
- A zone with AAAA records already loaded
- Root or sudo access

## Step 1: Configure IPv6 Listening

Edit `/etc/powerdns/pdns.conf`:

```bash
# IPv6 listen address (:: means all IPv6 interfaces)
local-address=0.0.0.0, ::

# Bind to port 53 explicitly
local-port=53

# Enable DNSSEC
enable-lua-records=yes
```

## Step 2: Enable DNSSEC on a Zone

PowerDNS uses `pdnsutil` to manage DNSSEC keys and policies:

```bash
# Secure the zone — PowerDNS generates KSK and ZSK automatically
sudo pdnsutil secure-zone example.com

# Verify the zone is secured and keys are created
sudo pdnsutil show-zone example.com
```

## Step 3: Review the Generated Keys

```bash
# List all DNSSEC keys for the zone
sudo pdnsutil list-keys example.com

# Output shows key IDs, algorithms, and key types (KSK/ZSK)
# Example output:
# example.com. 257 3 13 <KSK public key data>  # KSK
# example.com. 256 3 13 <ZSK public key data>  # ZSK
```

## Step 4: Rectify the Zone

After securing a zone, rectify it so all records have correct hashes and ordering:

```bash
# Rectify ensures NSEC/NSEC3 records are correct
sudo pdnsutil rectify-zone example.com

# Check for any issues
sudo pdnsutil check-zone example.com
```

## Step 5: Export DS Records for the Parent Zone

```bash
# Export the DS record to send to your registrar
sudo pdnsutil show-zone example.com | grep "DS:"

# Or export in zone-file format
sudo pdnsutil export-zone-ds example.com
```

## Step 6: Add AAAA Records via the API or CLI

```bash
# Add an AAAA record using pdnsutil
sudo pdnsutil add-record example.com www AAAA 300 "2001:db8:1::10"

# Add a record for the nameserver itself
sudo pdnsutil add-record example.com ns1 AAAA 300 "2001:db8:1::1"

# Rectify again after adding records
sudo pdnsutil rectify-zone example.com
```

## Step 7: Verify DNSSEC Over IPv6

```bash
# Query your PowerDNS server directly over IPv6
dig +dnssec AAAA www.example.com @2001:db8:1::1

# Check that the AD flag is set in the response
# Look for "RRSIG AAAA" in the answer section

# Use delv for thorough DNSSEC validation
delv @2001:db8:1::1 AAAA www.example.com
```

## Using the PowerDNS API

PowerDNS also provides a REST API for zone management, which is useful for automation:

```bash
# Enable the API in pdns.conf
# api=yes
# api-key=your-secret-key
# webserver=yes
# webserver-address=::

# Create a zone via the API
curl -H "X-API-Key: your-secret-key" \
     -H "Content-Type: application/json" \
     -X POST \
     http://[::1]:8081/api/v1/servers/localhost/zones \
     -d '{"name":"example.com.","kind":"Native","nameservers":["ns1.example.com."]}'
```

## Monitoring DNSSEC Health

Use [OneUptime](https://oneuptime.com) to monitor your PowerDNS server over IPv6, checking both DNS availability and DNSSEC validation status. Configure alerts for when RRSIG records are about to expire.

## Conclusion

PowerDNS simplifies DNSSEC management with `pdnsutil`. Ensure IPv6 listening is configured, secure your zones, rectify them, and publish DS records to complete the chain of trust. Regularly monitor key expiry and zone health.
