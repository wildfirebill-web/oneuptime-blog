# How to Configure IPsec IPv6 with Pre-Shared Keys

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPsec, PSK, strongSwan, Authentication

Description: Learn how to configure IPv6 IPsec authentication using pre-shared keys (PSK) with strongSwan and best practices for PSK management and security.

## Overview

Pre-shared keys (PSK) are the simplest form of IPsec authentication: both peers share the same secret string, which is used to derive authentication keys during IKEv2 negotiation. PSK is suitable for small deployments with a limited number of tunnels. For large-scale deployments, certificate-based authentication is preferred.

## Generating a Strong PSK

```bash
# Generate a 256-bit (32-byte) random PSK

openssl rand -base64 32
# Example output: K7mX3pQnY9vL2wR8sT4uJ6hB1cE0fI5dG+N/oA==

# Or generate a hex PSK
openssl rand -hex 32
# Example: 4a8b3c2e1f9d7a6b5c4d3e2f1a0b9c8d7e6f5a4b3c2d1e0f9a8b7c6d5e4f3a2b
```

PSK requirements:
- Minimum 20 characters
- Maximum entropy - use random generators, not phrases
- Different PSK for each tunnel
- Store securely (password manager, secrets vault)

## strongSwan PSK Configuration

### Method 1: Inline PSK in swanctl.conf

```text
# /etc/swanctl/conf.d/vpn-psk.conf
connections {
    gw1-to-gw2 {
        version = 2
        local_addrs  = 2001:db8:gw1::1
        remote_addrs = 2001:db8:gw2::1

        local {
            auth = psk
            id = gw1.example.com    ! Must match peer's remote id
        }
        remote {
            auth = psk
            id = gw2.example.com    ! Must match peer's local id
        }

        children {
            site-tunnel {
                local_ts  = 2001:db8:site1::/48
                remote_ts = 2001:db8:site2::/48
                mode = tunnel
                esp_proposals = aes256gcm128-prfsha256-ecp256
                start_action = start
            }
        }

        proposals = aes256-sha256-ecp256
    }
}

secrets {
    ike-gw1-gw2 {
        id-1 = gw1.example.com
        id-2 = gw2.example.com
        secret = "K7mX3pQnY9vL2wR8sT4uJ6hB1cE0fI5dGNopA=="
    }
}
```

### Method 2: Separate Secrets File

```bash
# Keep secrets in a separate file for security
chmod 600 /etc/swanctl/secrets.conf

# /etc/swanctl/secrets.conf
secrets {
    ike-gw1-gw2 {
        id-1 = gw1.example.com
        id-2 = gw2.example.com
        secret = "K7mX3pQnY9vL2wR8sT4uJ6hB1cE0fI5dGNopA=="
    }
    ike-gw1-gw3 {
        id-1 = gw1.example.com
        id-2 = gw3.example.com
        secret = "Different-PSK-For-Each-Tunnel-X9kL3mNpQ=="
    }
}
```

## Libreswan PSK Configuration

```text
# /etc/ipsec.secrets
@gw1.example.com @gw2.example.com : PSK "K7mX3pQnY9vL2wR8sT4uJ6hB1cE0fI5dGNopA=="

# IP-based PSK (alternative)
2001:db8:gw1::1 2001:db8:gw2::1 : PSK "K7mX3pQnY9vL2wR8sT4uJ6hB1cE0fI5dGNopA=="
```

## PSK Identity Matching

The PSK secret is selected by matching `id` values:

```text
Scenario: GW1 connects to GW2

GW1 sends:  IDi = gw1.example.com
GW2 looks up: Find secret matching id-1=gw1.example.com AND id-2=gw2.example.com
              or id-1=gw2.example.com AND id-2=gw1.example.com (order doesn't matter)

If no matching secret → AUTHENTICATION_FAILED
```

```bash
# Debugging PSK matching
# Enable cfg logging level 4 in charon.conf
# Look for: "selecting PSK for identity 'gw1.example.com'"
tail -f /var/log/charon.log | grep -i 'psk\|secret\|auth'
```

## PSK Security Considerations

### Risk: PSK Compromise

If a PSK is compromised, any attacker with the PSK can:
- Impersonate either gateway
- Decrypt captured historical traffic (no perfect forward secrecy at PSK level)
- Establish unauthorized tunnels

**Mitigation:** Combine PSK with X.509 certificates for mutual authentication.

### PSK Rotation

```bash
# PSK rotation procedure (zero-downtime):
# 1. Add new PSK alongside old PSK in secrets
# 2. Update remote peer to accept new PSK
# 3. Initiate rekey from initiator side
# 4. Verify new SA established
# 5. Remove old PSK

# Step 1: Add new PSK (both old and new in secrets)
secrets {
    ike-gw1-gw2-old {
        id-1 = gw1.example.com
        id-2 = gw2.example.com
        secret = "OldPSK..."
    }
    ike-gw1-gw2-new {
        id-1 = gw1-new.example.com  ! Different ID triggers new SA
        id-2 = gw2-new.example.com
        secret = "NewPSK..."
    }
}
```

### Using HashiCorp Vault for PSK Storage

```bash
# Store PSK in Vault
vault kv put secret/ipsec/gw1-gw2 psk="K7mX3pQnY9vL2wR8..."

# Retrieve at runtime via script
PSK=$(vault kv get -field=psk secret/ipsec/gw1-gw2)

# Generate swanctl secrets dynamically
cat > /etc/swanctl/conf.d/secrets-dynamic.conf << EOF
secrets {
    ike-gw1-gw2 {
        id-1 = gw1.example.com
        id-2 = gw2.example.com
        secret = "$PSK"
    }
}
EOF

swanctl --load-creds
```

## Summary

PSK authentication for IPv6 IPsec uses shared secrets referenced by identity (FQDN or IP). In strongSwan swanctl.conf, set `auth = psk` in local/remote blocks and define the secret in a `secrets{}` block with matching `id-1`/`id-2` values. Use separate PSK files with `chmod 600` for security. Generate PSKs with `openssl rand -base64 32` and use a unique PSK for each tunnel. For production, consider certificate authentication for scalability and rotate PSKs periodically (every 90 days minimum) by adding a new PSK and performing a controlled rekey.
