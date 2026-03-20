# How to Configure a Mobile IPv6 Home Agent on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mobile IPv6, Home Agent, Linux, UMIP, MIPv6, Configuration

Description: Configure a Linux server as a Mobile IPv6 Home Agent using the UMIP (USAGI Mobile IPv6) daemon to accept Binding Updates and tunnel traffic to Mobile Nodes.

## Introduction

The Home Agent is the cornerstone of a Mobile IPv6 deployment. This guide covers installing and configuring the UMIP daemon on Ubuntu/Debian Linux to provide Home Agent functionality.

## Prerequisites

- Linux server with kernel 4.x+ and IPv6 enabled
- Two IPv6 addresses: one for management, one as the HA address
- The home network prefix assigned to the HA interface

## Step 1: Install UMIP

```bash
# Install from package or source
sudo apt-get update
sudo apt-get install umip

# Or build from source
git clone https://github.com/openairinterface/umip.git
cd umip
./autogen.sh
./configure --enable-ha
make
sudo make install
```

## Step 2: Enable Required Kernel Features

```bash
# Enable IPv6 forwarding
echo "net.ipv6.conf.all.forwarding = 1" | \
  sudo tee -a /etc/sysctl.d/99-mipv6.conf

# Enable proxy NDP (needed for HA to intercept traffic for MN HoAs)
echo "net.ipv6.conf.eth0.proxy_ndp = 1" | \
  sudo tee -a /etc/sysctl.d/99-mipv6.conf

# Accept router advertisements (needed if HA is not a default router)
echo "net.ipv6.conf.eth0.accept_ra = 2" | \
  sudo tee -a /etc/sysctl.d/99-mipv6.conf

sudo sysctl -p /etc/sysctl.d/99-mipv6.conf
```

## Step 3: Configure the UMIP Home Agent

```bash
# /etc/mip6d.conf — Home Agent configuration

# Role: Home Agent
NodeConfig HA;

# HA interface — must be on the home network
Interface "eth0" {
    # HA address on the home link
    HaRestartAfterReboot enabled;
}

# Home network prefix served by this HA
# All MNs with HoAs in this prefix register here
HomeAgentAddress 2001:db8:home::1;
HomeAgentPreference 10;  # Higher = preferred HA
HomeAgentLifetime 300;   # Seconds advertised to MNs

# IPsec configuration for BU authentication
# Using manual keys (for testing — use IKEv2 in production)
UseMnHaIPsec enabled;

IPsecPolicySet {
    HomeAgentAddress 2001:db8:home::1;

    # Policy for MN1
    IPsecPolicy {
        MnAddress 2001:db8:home::100;
        Direction out;
        IPsecType ESP;
        ReqID 1;
    }
    IPsecPolicy {
        MnAddress 2001:db8:home::100;
        Direction in;
        IPsecType ESP;
        ReqID 2;
    }
}
```

## Step 4: Configure IPsec Security Associations

For testing with manual keys (use IKEv2/strongSwan in production):

```bash
# /etc/ipsec.conf — manual SA for MN-HA authentication
ip xfrm state add \
  src 2001:db8:home::100 \
  dst 2001:db8:home::1 \
  proto esp \
  spi 0x1001 \
  mode transport \
  auth hmac\(sha256\) 0x$(openssl rand -hex 32) \
  enc aes 0x$(openssl rand -hex 16)

ip xfrm policy add \
  src 2001:db8:home::100/128 \
  dst 2001:db8:home::1/128 \
  proto 135 \
  dir in \
  tmpl \
    src 2001:db8:home::100 \
    dst 2001:db8:home::1 \
    proto esp \
    mode transport
```

## Step 5: Start the Home Agent Daemon

```bash
# Start UMIP in HA mode
sudo mip6d -c /etc/mip6d.conf

# Or as a systemd service
sudo systemctl enable mip6d
sudo systemctl start mip6d

# Check status
sudo systemctl status mip6d
```

## Step 6: Verify Home Agent Operation

```bash
# View active bindings (MN registrations)
sudo mip6d -n

# Expected output:
# Home Agent Binding Cache:
# HoA: 2001:db8:home::100
# CoA: 2001:db8:foreign::50
# Lifetime: 587
# Flags: R (registered)

# Check proxy NDP entries (HA creates one per registered MN)
ip -6 neigh show proxy dev eth0

# Verify tunnel interfaces created by UMIP
ip tunnel show | grep mip

# Test by pinging the MN's HoA from the home network
ping6 -c 5 2001:db8:home::100
```

## Monitoring Home Agent Health

```bash
# Watch binding cache updates in real-time
watch -n5 "sudo mip6d -n"

# Monitor UMIP logs
journalctl -u mip6d -f

# Check for expired bindings
sudo mip6d -n | grep "Lifetime: 0"
```

## Conclusion

Configuring a Linux Home Agent with UMIP enables Mobile IPv6 for your network. Production deployments should replace manual IPsec keys with IKEv2/strongSwan for certificate-based authentication. Use OneUptime to monitor the HA's IPv6 address availability and binding update response times.
