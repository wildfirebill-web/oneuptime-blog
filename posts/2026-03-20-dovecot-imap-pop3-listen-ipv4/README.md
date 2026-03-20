# How to Configure Dovecot IMAP/POP3 to Listen on IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dovecot, IMAP, POP3, IPv4, Mail Server, Configuration

Description: Configure Dovecot to listen on specific IPv4 addresses for IMAP and POP3 connections, restricting mail client access to particular network interfaces.

## Introduction

Dovecot listens on all interfaces by default. Binding to specific IPv4 addresses restricts mail client access to designated network interfaces, improving security on multi-homed servers.

## Configuring listen Directive

```bash
# /etc/dovecot/dovecot.conf

# Listen on specific IPv4 address only
listen = 203.0.113.10

# Listen on multiple IPv4 addresses
listen = 203.0.113.10, 10.0.0.1

# Listen on all IPv4 interfaces (without IPv6)
listen = *   # This is the default (also listens on ::)

# To listen on IPv4 only (not IPv6):
listen = *    # Then disable IPv6 at OS level, or:
# listen = 0.0.0.0   # Explicit IPv4 wildcard
```

## Disabling IPv6 in Dovecot

```bash
# /etc/dovecot/dovecot.conf

# Only listen on IPv4 (use 0.0.0.0 wildcard)
listen = 0.0.0.0

# Alternatively, list specific IPv4 addresses
listen = 203.0.113.10

# Force IPv4 for outgoing connections (e.g., LMTP delivery)
# Set in protocol lmtp section:
```

## Port-Specific Listener Configuration

```bash
# /etc/dovecot/conf.d/10-master.conf

service imap-login {
    inet_listener imap {
        address = 203.0.113.10    # IMAP port 143
        port = 143
    }
    inet_listener imaps {
        address = 203.0.113.10    # IMAP SSL port 993
        port = 993
        ssl = yes
    }
}

service pop3-login {
    inet_listener pop3 {
        address = 203.0.113.10    # POP3 port 110
        port = 110
    }
    inet_listener pop3s {
        address = 203.0.113.10    # POP3 SSL port 995
        port = 995
        ssl = yes
    }
}

service managesieve-login {
    inet_listener sieve {
        address = 10.0.0.1        # Sieve on internal IP only
        port = 4190
    }
}
```

## SSL/TLS Configuration

```bash
# /etc/dovecot/conf.d/10-ssl.conf

ssl = required    # Require SSL for all connections
ssl_cert = </etc/letsencrypt/live/mail.example.com/fullchain.pem
ssl_key  = </etc/letsencrypt/live/mail.example.com/privkey.pem

# Modern TLS protocols only
ssl_min_protocol = TLSv1.2
ssl_cipher_list = ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
```

## Verifying Dovecot Listeners

```bash
# Check Dovecot configuration
sudo dovecot -n | grep -E "listen|address|port"

# Verify ports are listening on correct IP
sudo ss -tlnp | grep dovecot
# Expected:
# LISTEN 0 100 203.0.113.10:143  0.0.0.0:*  users:(("dovecot",...))
# LISTEN 0 100 203.0.113.10:993  0.0.0.0:*
# LISTEN 0 100 203.0.113.10:110  0.0.0.0:*
# LISTEN 0 100 203.0.113.10:995  0.0.0.0:*

# Test IMAP connection
openssl s_client -connect 203.0.113.10:993 -servername mail.example.com

# Reload Dovecot
sudo systemctl reload dovecot
```

## Conclusion

Dovecot's `listen` directive and `inet_listener` address parameters control which IPv4 interfaces serve IMAP and POP3. Set `listen = 203.0.113.10` for server-wide IPv4 restriction, or configure individual `address` fields per service in `10-master.conf` for granular control. Enable SSL on ports 993 and 995 with modern TLS protocols for all production mail servers.
