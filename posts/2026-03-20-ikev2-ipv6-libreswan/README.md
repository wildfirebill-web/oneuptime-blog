# How to Configure IKEv2 for IPv6 on Linux with Libreswan

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IKEv2, Libreswan, IPsec, VPN

Description: Learn how to configure IKEv2 for IPv6 VPNs on Linux using Libreswan, the successor to Openswan, with site-to-site and host-to-host configurations.

## Overview

Libreswan is an open-source IKEv1/IKEv2 implementation for Linux, commonly used in Red Hat-based distributions. It is the successor to Openswan and FreeS/WAN. For IPv6, Libreswan supports both site-to-site tunnels and host-to-host configurations using ipsec.conf or the modern ipsec.conf connection format.

## Installation

```bash
# RHEL/CentOS/Fedora

sudo dnf install libreswan

# Debian/Ubuntu
sudo apt install libreswan

# Initialize NSS database (required for certificates)
ipsec initnss

# Start service
sudo systemctl enable --now ipsec

# Verify
ipsec --version
```

## Site-to-Site IPv6 Configuration (PSK)

### /etc/ipsec.conf

```text
config setup
    logfile=/var/log/pluto.log
    logappend=no

conn %default
    ikelifetime=28800s
    keylife=3600s
    rekeymargin=540s
    keyingtries=3
    authby=secret
    keyexchange=ikev2

conn site1-to-site2
    left=2001:db8:gw1::1
    leftsubnet=2001:db8:net1::/48
    leftid=@gw1.example.com

    right=2001:db8:gw2::1
    rightsubnet=2001:db8:net2::/48
    rightid=@gw2.example.com

    esp=aes256gcm128-sha256-ecp256
    ike=aes256-sha256-ecp256
    ikev2=insist
    auto=start
```

### /etc/ipsec.secrets

```text
# PSK for site-to-site
@gw1.example.com @gw2.example.com : PSK "StrongSharedKey123ChangeThis!"
```

```bash
# Load configuration
ipsec auto --add site1-to-site2

# Start tunnel
ipsec auto --up site1-to-site2

# Check status
ipsec status
ipsec trafficstatus
```

## Host-to-Host IPv6 Transport Mode

```text
conn host-a-to-b
    left=2001:db8:1::1
    right=2001:db8:1::2
    leftid=@host-a.example.com
    rightid=@host-b.example.com
    type=transport
    esp=aes256gcm128-sha256-ecp256
    ike=aes256-sha256-ecp256
    ikev2=insist
    authby=secret
    auto=start
    compress=no
```

## Certificate Authentication

```bash
# Generate certificates using NSS
# Create CA
certutil -S -n "VPN CA" -s "CN=VPN CA" -x -t "CT,C,C" -v 60 -d /etc/ipsec.d

# Create GW1 certificate
certutil -S -n "gw1" -s "CN=gw1.example.com" -c "VPN CA" -t "u,u,u" -v 12 \
  -d /etc/ipsec.d -8 "2001:db8:gw1::1"

# Export and share CA cert to remote side
pk12util -o gw1.p12 -n "gw1" -d /etc/ipsec.d
```

```text
# Certificate-based connection
conn cert-site1-to-site2
    left=2001:db8:gw1::1
    leftsubnet=2001:db8:net1::/48
    leftid=@gw1.example.com
    leftcert=gw1

    right=2001:db8:gw2::1
    rightsubnet=2001:db8:net2::/48
    rightid=@gw2.example.com
    rightrsasigkey=%cert

    authby=rsasig
    ikev2=insist
    auto=start
```

## Verification and Monitoring

```bash
# Show all active tunnels
ipsec status

# Sample output:
# 006 #1: "site1-to-site2" state:ESTABLISHED; established 45s ago; IKEv2; SPI:... SPIr:...
# 004 #2: "site1-to-site2":1 ESP tunnel[1] ...

# Show traffic statistics
ipsec trafficstatus

# Show SA details
ipsec showstates

# Show kernel XFRM state
ip xfrm state list

# Ping across tunnel
ping6 2001:db8:net2::1

# tcpdump: Verify ESP traffic on gateway interface
tcpdump -i eth0 'ip6 proto 50' -n
```

## Troubleshooting

```bash
# Enable verbose debugging
ipsec pluto --nofork --debug-all 2>&1 | head -100

# Common issues:
# "AUTHENTICATE: Failed to verify IKEv2 AUTH payload"
# → Wrong PSK or cert mismatch - check /etc/ipsec.secrets

# "No matching policy" → check leftsubnet/rightsubnet match

# "unable to find compatible proposals"
# → Mismatched ike= or esp= proposals - check both sides match

# Restart after configuration change
systemctl restart ipsec
ipsec auto --add site1-to-site2
ipsec auto --up site1-to-site2
```

## Key Differences: Libreswan vs strongSwan

| Feature | Libreswan | strongSwan |
|---------|-----------|-----------|
| Config format | ipsec.conf | swanctl.conf |
| NSS integration | Yes (built-in) | Optional |
| Package availability | RHEL/Fedora default | Debian/Ubuntu common |
| Remote access VPN | Supported | Strong EAP support |
| Logging | /var/log/pluto.log | journald or file |

## Summary

Libreswan uses ipsec.conf with `conn` blocks defining left/right addresses, subnets, and authentication. For IPv6, set `left`/`right` to IPv6 addresses and `leftsubnet`/`rightsubnet` to IPv6 prefixes. Use `ikev2=insist` to enforce IKEv2 and `authby=secret` with PSK in `/etc/ipsec.secrets`. Start connections with `ipsec auto --up <conn>` and monitor with `ipsec status` and `ipsec trafficstatus`. Libreswan is the default IPsec implementation on RHEL/Fedora and integrates well with SELinux and system NSS certificate stores.
