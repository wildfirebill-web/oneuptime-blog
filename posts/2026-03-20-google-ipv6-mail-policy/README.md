# How to Meet Google IPv6 Mail Policy Requirements

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Email, IPv6, Gmail, Google, SPF, DKIM, DMARC, Mail Deliverability

Description: Meet Google's specific IPv6 mail policy requirements including PTR records, SPF, DKIM, and DMARC to ensure email from IPv6 servers reaches Gmail inboxes.

## Introduction

Google enforces strict requirements for email delivered from IPv6 addresses to Gmail. Failing these checks results in messages being rejected or spam-foldered with specific error codes. This guide covers every requirement Google mandates for IPv6 senders.

## Google's IPv6 Mail Requirements

Google requires the following for mail from IPv6 senders:

1. **PTR record**: The sending IPv6 address must have a valid PTR record
2. **Forward-confirmed reverse DNS (FCrDNS)**: The PTR hostname must have an AAAA record pointing back to the sending IP
3. **SPF**: The sending IP must be authorized in the domain's SPF record using `ip6:`
4. **DKIM**: Messages must be DKIM-signed
5. **DMARC**: The domain should have a DMARC policy
6. **Sending history**: New IPv6 addresses should warm up gradually

## Checking Your Configuration

Run this checklist against your mail server's IPv6 address and domain:

```bash
MAIL_IP="2001:db8::10"
MAIL_DOMAIN="example.com"
MAIL_HOST="mail.example.com"

# 1. Check PTR record
echo "=== PTR Record ==="
dig -x $MAIL_IP +short

# 2. Check FCrDNS (AAAA for PTR result)
echo "=== FCrDNS AAAA ==="
dig AAAA $MAIL_HOST +short

# 3. Check SPF record
echo "=== SPF Record ==="
dig TXT $MAIL_DOMAIN +short | grep spf

# 4. Check DMARC record
echo "=== DMARC Record ==="
dig TXT _dmarc.$MAIL_DOMAIN +short

# 5. Check DKIM selector (replace 'mail' with your selector)
echo "=== DKIM Record ==="
dig TXT mail._domainkey.$MAIL_DOMAIN +short
```

## Configuring PTR and FCrDNS

Your hosting provider must configure the reverse DNS for your IPv6 block:

```bash
# Verify PTR resolves to your mail hostname
dig -x 2001:db8::10 +short
# Must return: mail.example.com.

# Verify AAAA record resolves back
dig AAAA mail.example.com +short
# Must return: 2001:db8::10
```

## SPF Record with IPv6

Ensure your SPF record includes the `ip6:` mechanism:

```dns
; SPF TXT record
example.com.  300  IN  TXT  "v=spf1 ip4:203.0.113.10 ip6:2001:db8::10 ~all"
```

Verify SPF passes for your IPv6 address using an online checker or the `pyspf` library:

```bash
# Install spf checker
pip3 install pyspf

python3 -c "
import spf
result, _, msg = spf.check2(
    i='2001:db8::10',
    s='test@example.com',
    h='mail.example.com'
)
print(f'SPF result: {result}')
print(f'Message: {msg}')
"
```

## DKIM Configuration for IPv6 Servers

DKIM signing is independent of IP version — configure it in Postfix with OpenDKIM:

```bash
# Install OpenDKIM
sudo apt install -y opendkim opendkim-tools

# Generate DKIM key pair
sudo mkdir -p /etc/opendkim/keys/example.com
sudo opendkim-genkey -s mail -d example.com -D /etc/opendkim/keys/example.com/

# Publish the public key in DNS as a TXT record
sudo cat /etc/opendkim/keys/example.com/mail.txt
# Add the TXT record to your DNS zone
```

## DMARC Policy

Publish a DMARC policy to tell receivers how to handle SPF/DKIM failures:

```dns
; Start with monitoring mode (p=none)
_dmarc.example.com.  300  IN  TXT  "v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com"

; Progress to quarantine after monitoring
_dmarc.example.com.  300  IN  TXT  "v=DMARC1; p=quarantine; pct=100; rua=mailto:dmarc-reports@example.com"

; Enforce rejection for full protection
_dmarc.example.com.  300  IN  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc-reports@example.com"
```

## Testing Delivery to Gmail

Send a test email and check Google's feedback:

```bash
# Send a test email to a Gmail account you control
swaks --from sender@example.com \
      --to yourtest@gmail.com \
      --server [2001:db8::10]:25

# Check the received headers in Gmail:
# Received-SPF: pass
# DKIM-Signature: present
# Authentication-Results: ... dkim=pass; spf=pass; dmarc=pass
```

## Handling the Google Error: 550-5.7.1

If you receive `550-5.7.1 ... Our system has detected that this message does not meet IPv6 sending guidelines`:

```bash
# The error usually means PTR or FCrDNS is missing
# Verify with:
dig -x <your-IPv6> +short       # Should return your hostname
dig AAAA <your-hostname> +short # Should return your IPv6
```

## Conclusion

Meeting Google's IPv6 mail policy requires PTR with FCrDNS, SPF with `ip6:`, DKIM signing, and a DMARC policy. With all four in place and a warmed-up IPv6 address, mail to Gmail from IPv6 servers achieves excellent deliverability.
