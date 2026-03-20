# How to Configure IPsec with Pre-Shared Keys on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPsec, PSK, Pre-Shared Key, strongSwan, Linux, VPN

Description: Configure IPsec VPN authentication using pre-shared keys (PSK) on Linux with strongSwan, including secure key generation and secrets file management.

Pre-shared keys (PSK) are the simplest authentication method for IPsec. Both sides of the tunnel share the same secret key. While less scalable than certificate-based auth, PSK is easier to configure for a small number of site-to-site tunnels.

## Generating a Secure Pre-Shared Key

```bash
# Generate a strong PSK (32+ characters recommended)
openssl rand -base64 32
# Example output: 7mV8JkXq2nP4bR1hD9cF5wL3yN6tG0sI

# Or use /dev/urandom
dd if=/dev/urandom bs=32 count=1 2>/dev/null | base64 | tr -d '\n'

# Store it securely — never use weak PSKs
```

## Configuring PSK in ipsec.secrets

The `ipsec.secrets` file holds authentication credentials:

```conf
# /etc/ipsec.secrets

# Format: local_id remote_id : PSK "secret"

# Simple form — any peer can authenticate with this PSK
%any %any : PSK "your-very-long-random-secret-here"

# Specific peer PSK (more secure)
@gateway-a @gateway-b : PSK "specific-tunnel-secret"

# Using IP addresses as identifiers
1.2.3.4 5.6.7.8 : PSK "ip-based-psk-secret"
```

```bash
# CRITICAL: Secure the secrets file
sudo chmod 600 /etc/ipsec.secrets
sudo chown root:root /etc/ipsec.secrets
```

## Connection Configuration with PSK

```conf
# /etc/ipsec.conf

conn psk-tunnel
    keyexchange=ikev2
    auto=start
    type=tunnel

    # Local gateway
    left=%defaultroute
    leftid=@my-gateway
    leftsubnet=10.1.0.0/24
    # Authenticate using PSK
    leftauth=psk

    # Remote gateway
    right=5.6.7.8
    rightid=@remote-gateway
    rightsubnet=10.2.0.0/24
    rightauth=psk

    # Crypto settings
    ike=aes256-sha256-modp2048!
    esp=aes256-sha256!
```

## Multiple PSK Tunnels

For multiple site-to-site tunnels with different keys:

```conf
# /etc/ipsec.secrets

# Each tunnel can have its own PSK
@gateway-a @site-b : PSK "secret-for-site-b"
@gateway-a @site-c : PSK "secret-for-site-c"
@gateway-a @site-d : PSK "secret-for-site-d"
```

## Rotating PSKs

PSK rotation must be coordinated between both sides to avoid downtime:

```bash
# Step 1: Add the NEW key to ipsec.secrets alongside the old one
# Step 2: Update the remote peer's ipsec.secrets to the new key
# Step 3: Reload strongSwan (new connections use the new key)
# Step 4: When all tunnels have renegotiated, remove the old key

# Reload secrets without full restart
sudo ipsec rereadsecrets
```

## Verifying PSK Authentication

```bash
# Enable auth logging to see PSK verification
sudo ipsec stroke loglevel ike 3

# Bring up the tunnel
sudo ipsec up psk-tunnel

# Success log message:
# "authentication of ... with PSK successful"

# Failure log message:
# "AUTH_FAILED notify received ... PSK"
# → Check that PSK matches on both sides exactly (case-sensitive)
```

## Best Practices for PSK Security

```bash
# 1. Use long, random keys (32+ characters)
openssl rand -base64 48

# 2. Use per-tunnel unique keys, not one shared PSK
# 3. Protect the secrets file (chmod 600)
# 4. Rotate PSKs annually or after any compromise
# 5. Use certificate authentication for production if possible
# 6. Monitor for authentication failures as they may indicate PSK exposure
sudo journalctl -u strongswan | grep -i "auth.*fail"
```

PSK authentication is a pragmatic choice for two-site tunnels, but move to certificate authentication when managing more than 5-10 tunnels or when compliance requires it.
