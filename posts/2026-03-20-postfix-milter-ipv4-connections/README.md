# How to Configure Postfix milter for IPv4 Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, Milter, IPv4, Email, DKIM, SpamAssassin, Configuration, Security

Description: Learn how to configure Postfix milter integration over IPv4 TCP sockets to connect to mail filtering services like OpenDKIM and SpamAssassin.

---

Milters (mail filters) integrate with Postfix to process messages in real time — signing with DKIM, checking SPF, filtering spam, or running virus scans. Milters communicate via Unix sockets or TCP sockets; using a TCP socket on an IPv4 address allows milters to run on remote servers.

## Milter Connection Types

```
Unix socket:  unix:/run/opendkim/opendkim.sock
TCP socket:   inet:12301@127.0.0.1     (local IPv4)
TCP socket:   inet:12301@10.0.0.5      (remote IPv4)
```

## Configuring OpenDKIM as a Milter over TCP

### OpenDKIM Configuration

```ini
# /etc/opendkim.conf

# Listen on a TCP socket on the loopback IPv4 address
Socket inet:12301@127.0.0.1

# Signing mode
Mode sv

# Key and domain settings
Domain example.com
KeyFile /etc/opendkim/keys/example.com/default.private
Selector default
```

```bash
systemctl restart opendkim
ss -tlnp | grep 12301   # Verify OpenDKIM is listening
```

### Postfix Configuration

```ini
# /etc/postfix/main.cf

# IPv4 only
inet_protocols = ipv4

# --- Milter configuration ---
# Connect to OpenDKIM on IPv4 loopback
milter_default_action = accept         # Accept mail if milter is unavailable
milter_protocol = 6                    # Milter protocol version

# Specify the milter socket (inet:port@ip format)
smtpd_milters = inet:127.0.0.1:12301  # For incoming mail (SMTP)
non_smtpd_milters = inet:127.0.0.1:12301  # For locally submitted mail

# Timeout settings for the milter connection
milter_connect_timeout = 30s
milter_command_timeout = 30s
milter_content_timeout = 300s
```

## Adding SpamAssassin as a Second Milter

```bash
# Install spamass-milter (bridges SpamAssassin to the milter protocol)
apt install spamass-milter -y

# /etc/default/spamass-milter
OPTIONS="-u spamass-milter -i 127.0.0.1 -- -u spamd -d localhost"
SOCKET="inet:783@127.0.0.1"
```

```ini
# /etc/postfix/main.cf

# Chain multiple milters separated by spaces
smtpd_milters = inet:127.0.0.1:12301 inet:127.0.0.1:783
non_smtpd_milters = inet:127.0.0.1:12301
```

## Remote Milter on Another IPv4 Server

```ini
# Connect to a milter running on a different server in the network
smtpd_milters = inet:10.0.0.50:12301
```

Ensure the milter is configured to accept connections from the Postfix server's IPv4 address and that the firewall permits the connection.

## Verifying Milter Connections

```bash
# Test Postfix config
postfix check

# Reload Postfix
systemctl reload postfix

# Send a test email and check headers for DKIM signature
echo "Test milter" | sendmail -v user@example.com

# Check mail log for milter activity
tail -f /var/log/mail.log | grep milter
```

## Key Takeaways

- Use `inet:port@ip` format to connect to milters over IPv4 TCP instead of Unix sockets.
- `milter_default_action = accept` prevents mail from being rejected if the milter crashes.
- Multiple milters can be chained with space-separated entries in `smtpd_milters`.
- Remote milters on other IPv4 servers require firewall rules to allow the connection from the Postfix host.
