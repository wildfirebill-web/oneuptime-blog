# How to Configure ProFTPD TLS Encryption on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ProFTPD, TLS, FTP, IPv4, FTPS, Security, Configuration

Description: Learn how to configure ProFTPD with TLS (FTPS) on an IPv4 server to encrypt FTP control and data connections.

---

Plain FTP transmits credentials and data in cleartext. FTPS (FTP over TLS) encrypts the connection using the same TLS protocol used by HTTPS. ProFTPD's `mod_tls` module enables FTPS with minimal configuration.

## Prerequisites

```bash
# Enable mod_tls module (usually bundled with ProFTPD)

apt install proftpd-mod-crypto -y  # Debian/Ubuntu
```

## Generating a Certificate

```bash
# Generate a self-signed certificate (or use Let's Encrypt in production)
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /etc/proftpd/ssl/proftpd.key \
  -out    /etc/proftpd/ssl/proftpd.crt \
  -subj   "/CN=ftp.example.com/O=My Org/C=US"

# Set secure permissions
chmod 600 /etc/proftpd/ssl/proftpd.key
chown proftpd:proftpd /etc/proftpd/ssl/proftpd.key
```

## ProFTPD TLS Configuration

```apache
# /etc/proftpd/proftpd.conf

# Load the TLS module
LoadModule mod_tls.c

# --- Bind to IPv4 ---
Port       21
ServerName "Secure FTP Server"

# --- TLS configuration ---
<IfModule mod_tls.c>
    # Enable TLS support
    TLSEngine on

    # Log TLS events (0=none, 1=connections, 3=verbose)
    TLSLog /var/log/proftpd/tls.log

    # Require TLS for all connections (use 'on' to make it optional)
    TLSRequired on

    # Certificate and key paths
    TLSRSACertificateFile    /etc/proftpd/ssl/proftpd.crt
    TLSRSACertificateKeyFile /etc/proftpd/ssl/proftpd.key

    # Disable weak protocols
    TLSProtocol TLSv1.2 TLSv1.3

    # Strong cipher suites
    TLSCipherSuite HIGH:!aNULL:!MD5:!3DES

    # Require clients to present a certificate (for mutual TLS; optional)
    # TLSVerifyClient on

    # Allow data connections to reuse the TLS session from the control connection
    TLSOptions NoCertRequest EnableDiags NoSessionReuseRequired

    # If TLS handshake fails, do not fall back to cleartext
    TLSRenegotiate none
</IfModule>

# --- Passive mode settings ---
PassivePorts 40000 50000
MasqueradeAddress ftp.example.com
```

## Making TLS Optional (Allow Both Plain and TLS)

```apache
<IfModule mod_tls.c>
    TLSEngine on
    TLSRequired off   # Don't require TLS; allow plain FTP too (not recommended)
    TLSRSACertificateFile    /etc/proftpd/ssl/proftpd.crt
    TLSRSACertificateKeyFile /etc/proftpd/ssl/proftpd.key
    TLSProtocol TLSv1.2 TLSv1.3
</IfModule>
```

## Applying the Configuration

```bash
# Test config
proftpd --configtest

# Restart ProFTPD
systemctl restart proftpd

# Verify ProFTPD is listening on port 21
ss -tlnp | grep proftpd
```

## Testing FTPS

```bash
# Test with curl (explicit FTPS via STARTTLS on port 21)
curl --ftp-ssl ftp://ftp.example.com/ --user user:password

# Test with lftp (interactive FTPS client)
lftp -e "set ftp:ssl-force true; ls; quit" -u user,password ftp.example.com
```

## Key Takeaways

- `TLSRequired on` forces all clients to negotiate TLS; `TLSRequired off` makes it optional.
- `TLSProtocol TLSv1.2 TLSv1.3` disables weak SSL/TLS versions.
- `NoSessionReuseRequired` is often needed for compatibility with Windows FTP clients.
- Use `TLSLog` to debug TLS handshake issues when clients can't connect.
