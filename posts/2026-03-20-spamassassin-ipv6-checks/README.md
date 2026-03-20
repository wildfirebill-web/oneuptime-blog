# How to Configure SpamAssassin for IPv6 Checks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SpamAssassin, IPv6, Email, Spam Filtering, Mail Server, DNSBL

Description: Configure SpamAssassin to correctly handle IPv6 sender addresses in trusted networks, DNSBL checks, and whitelist rules for accurate spam scoring.

## Introduction

SpamAssassin performs many checks based on the sender's IP address, including DNSBL lookups, trusted network detection, and whitelist matching. When mail arrives from or through IPv6 addresses, SpamAssassin needs proper configuration to avoid false positives and ensure accurate spam detection.

## Installing SpamAssassin

```bash
# Ubuntu/Debian

sudo apt update && sudo apt install -y spamassassin spamc

# Enable and start the daemon
sudo systemctl enable --now spamassassin
```

## Configuring Trusted Networks with IPv6

SpamAssassin's `trusted_networks` and `internal_networks` affect how relays are evaluated. IPv6 addresses use standard CIDR notation:

```bash
sudo tee /etc/spamassassin/local.cf << 'EOF'
# Trust IPv4 and IPv6 internal networks
trusted_networks 127.0.0.0/8 ::1/128
trusted_networks 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16
trusted_networks 2001:db8::/32 fd00::/8

# Define internal networks (relays you control)
internal_networks 127.0.0.0/8 ::1/128
internal_networks 10.0.0.0/8
internal_networks 2001:db8::/32
EOF
```

## Configuring IPv6 Whitelist Entries

Whitelist specific IPv6 sender addresses or ranges:

```bash
sudo tee -a /etc/spamassassin/local.cf << 'EOF'
# Whitelist specific IPv6 senders
whitelist_from_rcvd *@trusted.example.com 2001:db8::10
whitelist_from_rcvd *@trusted.example.com [2001:db8::10]

# Add IP-based whitelist rule
# score will be applied if sender IP matches
EOF
```

## Configuring DNSBL for IPv6

SpamAssassin's DNSBL plugin supports IPv6 natively. The DNS lookup format differs for IPv6:

```bash
# Check which DNSBL plugins are active
grep -r "URIBL\|DNSBL\|RBL" /etc/spamassassin/ | grep "^[^#]"

# Add an IPv6-compatible DNSBL check in local.cf
sudo tee -a /etc/spamassassin/local.cf << 'EOF'
# IPv6-aware DNSBL check
header   RCVD_IN_XBL     eval:check_rbl('xbl', 'xbl.spamhaus.org.')
describe RCVD_IN_XBL     Received via a relay in Spamhaus XBL
tflags   RCVD_IN_XBL     net
score    RCVD_IN_XBL     2.0
EOF
```

## Updating SpamAssassin Rules

```bash
# Update rule sets (includes IPv6-related rules)
sudo sa-update
sudo systemctl restart spamassassin

# Run sa-update in verbose mode to see what's updated
sudo sa-update --debug 2>&1 | grep -i "ipv6\|update"
```

## Testing SpamAssassin with an IPv6 Message

Create a test email with an IPv6 Received header:

```bash
# Create a test email simulating IPv6 relay
cat > /tmp/test-ipv6.eml << 'EOF'
Received: from mail.example.com ([2001:db8::10])
  by mx.test.com with ESMTP id abc123;
  Thu, 19 Mar 2026 12:00:00 +0000
From: sender@example.com
To: recipient@test.com
Subject: IPv6 SpamAssassin Test
Message-ID: <test-ipv6@mail.example.com>
Date: Thu, 19 Mar 2026 12:00:00 +0000

Test message from IPv6 sender.
EOF

# Process with SpamAssassin
spamassassin -t < /tmp/test-ipv6.eml

# Check specific scores
spamassassin -t -D all < /tmp/test-ipv6.eml 2>&1 | grep -E "IPv6|trusted|whitelist"
```

## Checking How SpamAssassin Sees IPv6 Addresses

```bash
# Use spamassassin debug mode to see IP address handling
spamassassin -t -D bayes,dns < /tmp/test-ipv6.eml 2>&1 | grep -i "ip\|relay\|rdns"

# Check trusted_networks evaluation
spamassassin -t -D network < /tmp/test-ipv6.eml 2>&1 | grep "trusted\|internal"
```

## Handling spamd for IPv6 Connections

If using spamd as a daemon, ensure it accepts connections from IPv6 clients:

```bash
# Edit the spamd options in /etc/default/spamassassin
sudo nano /etc/default/spamassassin

# Set OPTIONS to bind on IPv6
OPTIONS="--create-prefs --max-children 5 --username spamd --helper-home-dir /var/lib/spamassassin -s /var/log/spamd.log --listen :: --listen 0.0.0.0"

sudo systemctl restart spamassassin

# Verify spamd is listening on IPv6
ss -tlnp | grep 783
```

## Conclusion

SpamAssassin IPv6 configuration centers on three areas: defining `trusted_networks` and `internal_networks` with IPv6 CIDR blocks, configuring IPv6-aware DNSBL checks, and ensuring spamd listens on IPv6 if used as a daemon. With these changes, SpamAssassin accurately evaluates IPv6 mail without incorrectly penalizing legitimate internal senders.
