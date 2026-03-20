# How to Set Up OpenVPN Server for IPv4 on pfSense

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: pfSense, OpenVPN, IPv4, VPN, Remote Access, Security

Description: Configure an OpenVPN remote access server on pfSense for IPv4, including certificate authority creation, server setup, client export, and firewall rules.

## Introduction

pfSense includes a full OpenVPN server implementation accessible through the GUI. The OpenVPN Wizard simplifies setup by creating the CA, server certificate, and basic configuration in a guided flow.

## Step 1: Create Certificate Authority

Navigate to **System > Cert. Manager > CAs > Add**:
- Method: Create an internal Certificate Authority
- Descriptive name: `pfSense-CA`
- Key type: RSA 4096
- Digest: SHA-256
- Lifetime: 3650 days

## Step 2: Create Server Certificate

Navigate to **System > Cert. Manager > Certificates > Add**:
- Method: Create an internal Certificate
- Descriptive name: `openvpn-server`
- Certificate Authority: `pfSense-CA`
- Certificate Type: Server Certificate

## Step 3: OpenVPN Server Setup

Navigate to **VPN > OpenVPN > Servers > Add**:

```text
Server Mode:         Remote Access (SSL/TLS + User Auth)
Protocol:            UDP on IPv4 only
Interface:           WAN
Local port:          1194
TLS Configuration:   Use TLS Key - Generate
Peer Certificate Authority: pfSense-CA
Server Certificate:  openvpn-server
DH Parameter Length: 4096 bit
Encryption Algorithm: AES-256-GCM
Auth digest algorithm: SHA256

Tunnel Network:      10.8.0.0/24
Redirect Gateway:    Force all client-generated IPv4 traffic through VPN
Local Network:       192.168.1.0/24
```

## Step 4: Firewall Rules

Navigate to **Firewall > Rules > WAN > Add**:
```text
Protocol: UDP
Destination: WAN address
Destination port: 1194
Description: Allow OpenVPN

```

Navigate to **Firewall > Rules > OpenVPN > Add**:
```text
Action: Pass
Source: any
Destination: any
Description: Allow all VPN traffic
```

## Step 5: Install OpenVPN Client Export Package

Navigate to **System > Package Manager > Available Packages**:
- Install: `openvpn-client-export`

Navigate to **VPN > OpenVPN > Client Export**:
- Download configuration for Windows/macOS/Linux clients

## Step 6: User Management

Navigate to **System > User Manager > Add**:
- Username: `vpnuser1`
- Password: `SecurePass!`
- Certificate: Create certificate for user (check box)

## Client Configuration (.ovpn extract)

```text
client
dev tun
proto udp4
remote 203.0.113.1 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
tls-version-min 1.2
cipher AES-256-GCM
auth SHA256
verb 3
```

## Conclusion

pfSense OpenVPN server setup involves creating a CA and server certificate, configuring the OpenVPN server with a tunnel subnet, adding WAN and OpenVPN firewall rules, and exporting client configurations. The `openvpn-client-export` package simplifies distributing ready-to-use `.ovpn` files to remote users.
