# How to Configure Postfix mynetworks for IPv4 Trusted Subnets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, Mynetworks, IPv4, SMTP Relay, Trusted Networks, Email

Description: Configure Postfix mynetworks to define trusted IPv4 subnets that are allowed to relay mail without authentication, controlling which servers can use your SMTP relay.

## Introduction

`mynetworks` defines the IPv4 addresses and subnets that Postfix trusts as "local" clients, allowing them to relay mail without authentication. Misconfiguring this to be too broad creates an open relay that spammers will exploit.

## Setting mynetworks

```bash
# /etc/postfix/main.cf

# Only allow localhost and specific trusted subnets

mynetworks = 127.0.0.0/8, 10.0.0.0/8, 192.168.1.0/24

# Or list specific hosts only
mynetworks = 127.0.0.1, 10.0.0.5, 10.0.0.6, 10.0.0.7

# NEVER do this (creates open relay):
# mynetworks = 0.0.0.0/0
```

## Understanding mynetworks_style

Alternative to explicit `mynetworks`:

```bash
# /etc/postfix/main.cf

# Let Postfix auto-detect local networks
mynetworks_style = subnet   # Trust all IPs on same subnet as server
# mynetworks_style = host     # Trust only 127.0.0.1
# mynetworks_style = class    # Trust by IP class (A/B/C) - avoid this

# Or override auto-detection with explicit list:
mynetworks = 127.0.0.0/8 [::1]/128
```

## Testing for Open Relay

```bash
# CRITICAL: test that non-trusted IPs CANNOT relay
# From an external host:
telnet 203.0.113.10 25
EHLO test.example.com
MAIL FROM:<attacker@evil.com>
RCPT TO:<victim@external-domain.com>

# Expected response from Postfix:
# 554 5.7.1 <victim@external-domain.com>: Relay access denied

# From a trusted network (should succeed):
# telnet 10.0.0.5 25
# ... should be allowed to send
```

## mynetworks with Hash File for Large Lists

For many trusted IPs:

```bash
# /etc/postfix/mynetworks_list
# One IP/CIDR per line
127.0.0.0/8
10.0.0.0/8
172.16.0.0/12
192.168.1.0/24
203.0.113.20    # Trusted partner server
```

```bash
# /etc/postfix/main.cf
mynetworks = cidr:/etc/postfix/mynetworks_list
```

## Combining mynetworks with SASL Auth

Allow authenticated users even if not in mynetworks:

```bash
# /etc/postfix/main.cf

# Trusted IP ranges (can relay without auth)
mynetworks = 127.0.0.0/8, 10.0.0.0/8

# Allow relay if in mynetworks OR if SASL auth succeeds
smtpd_relay_restrictions =
    permit_mynetworks
    permit_sasl_authenticated
    reject_unauth_destination

# The order matters:
# 1. Allow if in mynetworks
# 2. Allow if SASL authenticated
# 3. Reject anything else going to external domains
```

## Verifying mynetworks Configuration

```bash
# Check effective mynetworks setting
postconf mynetworks

# Check that the relay restriction is correct
postconf smtpd_relay_restrictions

# View Postfix map tables
postconf -m  # List supported map types
postmap -q "10.0.0.5" cidr:/etc/postfix/mynetworks_list

# Check mail log for relay decisions
sudo tail -f /var/log/mail.log | grep "relay access\|NOQUEUE"
```

## Conclusion

`mynetworks` defines which IPv4 addresses can relay mail through Postfix without authentication. Keep it minimal-`127.0.0.0/8` plus your internal network ranges. Always test from an external IP to confirm relay is rejected, and combine with `permit_sasl_authenticated` for authenticated users who may connect from outside trusted networks. Running an open relay results in blacklisting and mail delivery failures.
