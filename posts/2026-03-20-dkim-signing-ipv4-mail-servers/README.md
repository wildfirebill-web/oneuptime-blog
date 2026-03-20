# How to Set Up DKIM Signing for Mail Sent from IPv4 Servers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DKIM, IPv4, Postfix, Email Security, OpenDKIM, Authentication

Description: Configure OpenDKIM with Postfix to cryptographically sign outbound email from IPv4 servers, improving deliverability and preventing email spoofing.

## Introduction

DKIM (DomainKeys Identified Mail) adds a cryptographic signature to outgoing emails. Receiving servers verify the signature against a public key published in DNS, confirming the mail came from an authorized server and was not tampered with in transit.

## Installing and Configuring OpenDKIM

```bash
# Install OpenDKIM
sudo apt install opendkim opendkim-tools

# Generate DKIM key pair
sudo mkdir -p /etc/opendkim/keys/example.com
sudo opendkim-genkey -b 2048 -d example.com -D /etc/opendkim/keys/example.com -s mail -v

# Files created:
# /etc/opendkim/keys/example.com/mail.private  (keep secret)
# /etc/opendkim/keys/example.com/mail.txt      (DNS TXT record content)

# Set permissions
sudo chown -R opendkim:opendkim /etc/opendkim/keys/
sudo chmod 600 /etc/opendkim/keys/example.com/mail.private
```

## OpenDKIM Configuration

```bash
# /etc/opendkim.conf

# Signing mode (sign outgoing, verify incoming)
Mode            sv

# Signing key configuration
KeyTable        refile:/etc/opendkim/KeyTable
SigningTable    refile:/etc/opendkim/SigningTable
InternalHosts   refile:/etc/opendkim/TrustedHosts

# Socket for Postfix integration
Socket          local:/var/spool/postfix/opendkim/opendkim.sock
# Or TCP socket
# Socket         inet:8891@127.0.0.1

# Log settings
Syslog          yes
SyslogSuccess   yes
LogWhy          yes

# Canonicalization (relaxed handles minor whitespace changes)
Canonicalization relaxed/relaxed

# Required headers
OversignHeaders From
```

```bash
# /etc/opendkim/KeyTable
# Selector._domainkey.domain  domain:selector:/path/to/key
mail._domainkey.example.com  example.com:mail:/etc/opendkim/keys/example.com/mail.private
```

```bash
# /etc/opendkim/SigningTable
# Email pattern  →  key table entry
*@example.com   mail._domainkey.example.com
```

```bash
# /etc/opendkim/TrustedHosts
# These IPs are treated as internal (sign their mail)
127.0.0.1
203.0.113.10     # Your mail server IPv4
10.0.0.0/8       # Internal network
```

## Integrating OpenDKIM with Postfix

```bash
# Create socket directory with correct permissions
sudo mkdir -p /var/spool/postfix/opendkim
sudo chown opendkim:postfix /var/spool/postfix/opendkim

# /etc/postfix/main.cf
milter_default_action = accept
milter_protocol = 6
smtpd_milters = local:/opendkim/opendkim.sock
non_smtpd_milters = local:/opendkim/opendkim.sock
```

```bash
# Start OpenDKIM and reload Postfix
sudo systemctl enable --now opendkim
sudo systemctl reload postfix
```

## Publishing the DNS Record

```bash
# View the DNS TXT record to publish
cat /etc/opendkim/keys/example.com/mail.txt

# Content will be something like:
# mail._domainkey IN TXT "v=DKIM1; h=sha256; k=rsa; p=MIIBIjANBg..."

# Add to your DNS (example for BIND):
# mail._domainkey.example.com. IN TXT "v=DKIM1; h=sha256; k=rsa; p=..."
```

## Verifying DKIM

```bash
# Send a test email and check headers
echo "DKIM test" | mail -s "DKIM Test" test@gmail.com

# Check mail log for DKIM signature
sudo tail -f /var/log/mail.log | grep DKIM

# Verify online: send mail to check-auth2@verifier.port25.com
echo "test" | mail -s "DKIM verify" check-auth2@verifier.port25.com

# Query DKIM public key
dig TXT mail._domainkey.example.com
```

## Conclusion

OpenDKIM signs outgoing email with a private key and publishes the public key in DNS. The key steps are: generate a key pair with `opendkim-genkey`, configure `KeyTable`/`SigningTable`/`TrustedHosts`, integrate with Postfix via milter socket, and publish the public key to DNS. Verify with an online checker or by sending to Port25's verifier.
