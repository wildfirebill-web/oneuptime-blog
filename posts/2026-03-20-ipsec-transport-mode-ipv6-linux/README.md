# How to Configure IPsec Transport Mode with IPv6 on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPsec, Linux, Transport Mode, Security

Description: Step-by-step guide to configuring IPsec transport mode for IPv6 on Linux using both manual ip xfrm commands and strongSwan for host-to-host authentication and encryption.

## Overview

IPsec transport mode for IPv6 on Linux protects traffic between two specific IPv6 hosts. The original IPv6 source and destination addresses are preserved, but the payload is encrypted (ESP) and/or authenticated (AH). This is ideal for securing communication between specific servers on a trusted network.

## Prerequisites

```bash
# Verify kernel IPsec support
modinfo esp6
modinfo ah6
modinfo xfrm6_tunnel

# Load modules if needed
modprobe esp6
modprobe ah6
```

## Method 1: Manual ip xfrm Configuration

### Host A: 2001:db8:1::1
### Host B: 2001:db8:1::2

```bash
# ============================================================
# On BOTH hosts: create SAs and policies
# ============================================================

# On Host A: Create outbound SA (A→B)
ip xfrm state add \
  src 2001:db8:1::1 \
  dst 2001:db8:1::2 \
  proto esp \
  spi 0xABC123 \
  mode transport \
  aead "rfc4106(gcm(aes))" \
  0x0102030405060708090a0b0c0d0e0f100102030405 128

# On Host A: Create inbound SA (B→A)
ip xfrm state add \
  src 2001:db8:1::2 \
  dst 2001:db8:1::1 \
  proto esp \
  spi 0xDEF456 \
  mode transport \
  aead "rfc4106(gcm(aes))" \
  0x1112131415161718191a1b1c1d1e1f201112131415 128

# On Host A: Create outbound policy
ip xfrm policy add \
  src 2001:db8:1::1/128 \
  dst 2001:db8:1::2/128 \
  proto tcp \
  dir out \
  tmpl \
    src 2001:db8:1::1 \
    dst 2001:db8:1::2 \
    proto esp \
    mode transport

# On Host A: Create inbound policy
ip xfrm policy add \
  src 2001:db8:1::2/128 \
  dst 2001:db8:1::1/128 \
  proto tcp \
  dir in \
  tmpl \
    src 2001:db8:1::2 \
    dst 2001:db8:1::1 \
    proto esp \
    mode transport
```

```bash
# On Host B: Mirror of Host A's configuration
# Inbound SA on B = Outbound SA from A (same SPI)
ip xfrm state add \
  src 2001:db8:1::1 \
  dst 2001:db8:1::2 \
  proto esp \
  spi 0xABC123 \
  mode transport \
  aead "rfc4106(gcm(aes))" \
  0x0102030405060708090a0b0c0d0e0f100102030405 128

ip xfrm state add \
  src 2001:db8:1::2 \
  dst 2001:db8:1::1 \
  proto esp \
  spi 0xDEF456 \
  mode transport \
  aead "rfc4106(gcm(aes))" \
  0x1112131415161718191a1b1c1d1e1f201112131415 128

ip xfrm policy add \
  src 2001:db8:1::2/128 dst 2001:db8:1::1/128 \
  proto tcp dir out \
  tmpl src 2001:db8:1::2 dst 2001:db8:1::1 proto esp mode transport

ip xfrm policy add \
  src 2001:db8:1::1/128 dst 2001:db8:1::2/128 \
  proto tcp dir in \
  tmpl src 2001:db8:1::1 dst 2001:db8:1::2 proto esp mode transport
```

## Verification

```bash
# Verify SAs are installed
ip xfrm state list

# Verify policies are installed
ip xfrm policy list

# Test: send TCP traffic and capture — should show ESP packets
# On Host A:
tcpdump -i eth0 'ip6 proto 50' -n &
ssh 2001:db8:1::2

# tcpdump should show ESP traffic (protocol 50), not plaintext TCP
```

## Method 2: strongSwan Transport Mode

### /etc/swanctl/conf.d/transport.conf on Host A

```
connections {
    host-a-to-b {
        version = 2
        local_addrs  = 2001:db8:1::1
        remote_addrs = 2001:db8:1::2

        local {
            auth = psk
            id = 2001:db8:1::1
        }
        remote {
            auth = psk
            id = 2001:db8:1::2
        }

        children {
            transport-tcp {
                local_ts  = 2001:db8:1::1/128[tcp]
                remote_ts = 2001:db8:1::2/128[tcp]
                mode = transport
                esp_proposals = aes256gcm128-prfsha256-ecp256
                start_action = trap
            }
        }

        proposals = aes256-sha256-ecp256
    }
}

secrets {
    ike-host-a-b {
        id-a = 2001:db8:1::1
        id-b = 2001:db8:1::2
        secret = "ChangeThisToAStrongKey!"
    }
}
```

```bash
# Load and start
swanctl --load-all
swanctl --initiate child:transport-tcp

# Verify connection
swanctl --list-sas
```

## Persistence: Save xfrm State

Manual `ip xfrm` configs are not persistent. To restore on boot:

```bash
# Save
ip xfrm state list > /etc/ipsec-xfrm-state.conf
ip xfrm policy list > /etc/ipsec-xfrm-policy.conf

# Create systemd service to restore
cat > /etc/systemd/system/xfrm-restore.service << 'EOF'
[Unit]
Description=Restore IPsec XFRM state
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ip xfrm state flush
ExecStart=/sbin/ip xfrm policy flush
ExecStart=/bin/sh -c 'ip xfrm state restore < /etc/ipsec-xfrm-state.conf'
ExecStart=/bin/sh -c 'ip xfrm policy restore < /etc/ipsec-xfrm-policy.conf'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
systemctl enable xfrm-restore
```

## Summary

IPsec transport mode for IPv6 on Linux uses `ip xfrm state` (SAs) and `ip xfrm policy` (SPD) commands. Configure matching SA pairs on both hosts (same SPI, key, and algorithm). Use AES-GCM (AEAD) for simplicity and performance. strongSwan with swanctl provides automated IKEv2 negotiation and is preferred over manual xfrm configuration for production use. Verify with `ip xfrm state list` and `tcpdump -i eth0 'ip6 proto 50'` to confirm traffic is encrypted.
