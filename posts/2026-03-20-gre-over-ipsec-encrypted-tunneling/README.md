# How to Configure GRE over IPsec for Encrypted Tunneling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, GRE, IPsec, StrongSwan, Tunnel, VPN, Encryption, Networking

Description: Combine GRE tunnels with IPsec transport mode to create encrypted GRE tunnels that provide both multi-protocol encapsulation and cryptographic security.

## Introduction

GRE tunnels carry traffic efficiently but provide no encryption. IPsec in transport mode can encrypt GRE traffic, combining GRE's protocol flexibility with IPsec's security. This is a common enterprise VPN pattern: GRE handles routing and encapsulation, IPsec provides authentication and encryption.

## Architecture

```
Plaintext Layer:  [IP][GRE][Inner IP][Payload]
IPsec Transport:  [IP][ESP/AH][GRE][Inner IP][Payload]
```

## Prerequisites

- StrongSwan installed on both hosts (`apt install strongswan`)
- GRE tunnel planned between hosts (10.0.0.1 and 10.0.0.2)

## Step 1: Configure IPsec (StrongSwan)

### /etc/ipsec.conf on Host A

```
# /etc/ipsec.conf
config setup
    charondebug="ike 2, knl 2, cfg 2"

conn gre-ipsec
    type=transport
    left=10.0.0.1
    right=10.0.0.2
    leftprotoport=gre
    rightprotoport=gre
    ike=aes256-sha256-modp2048!
    esp=aes256-sha256!
    authby=secret
    auto=start
```

### /etc/ipsec.secrets

```
# /etc/ipsec.secrets
10.0.0.1 10.0.0.2 : PSK "your-strong-preshared-key-here"
```

Apply the same configuration on Host B with left/right swapped.

## Step 2: Start IPsec

```bash
# Start strongswan/charon
systemctl start strongswan

# Check status
ipsec status

# Look for the GRE SA (Security Association)
ipsec statusall | grep gre-ipsec
```

## Step 3: Create the GRE Tunnel

After IPsec is established, create the GRE tunnel as normal:

```bash
# Host A
ip tunnel add gre0 mode gre local 10.0.0.1 remote 10.0.0.2 ttl 255
ip addr add 172.16.0.1/30 dev gre0
ip link set gre0 up

# Host B
ip tunnel add gre0 mode gre local 10.0.0.2 remote 10.0.0.1 ttl 255
ip addr add 172.16.0.2/30 dev gre0
ip link set gre0 up
```

## Step 4: Verify Encryption

```bash
# Generate tunnel traffic
ping -c 5 172.16.0.2

# Verify IPsec is encrypting GRE traffic
ipsec statusall

# Capture on eth0 — should see ESP packets (not plain GRE)
tcpdump -i eth0 esp -n
# Should show: ESP encrypted packets, not readable GRE content
```

## Add Routes Through the Encrypted Tunnel

```bash
# On Host A: route to Host B's LAN
ip route add 192.168.2.0/24 via 172.16.0.2

# On Host B: route to Host A's LAN
ip route add 192.168.1.0/24 via 172.16.0.1
```

## Conclusion

GRE over IPsec combines GRE's routing flexibility with IPsec's encryption. Use IPsec in transport mode (`type=transport`) to encrypt only the GRE protocol traffic between the two hosts. The GRE tunnel is configured exactly as a normal unencrypted tunnel, but all GRE packets are automatically encrypted by the IPsec SA. This is a proven VPN architecture used in enterprise networks.
