# How to Install and Configure an OpenVPN Server with IPv4 on Ubuntu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenVPN, VPN, IPv4, Ubuntu, Linux, Networking

Description: Install OpenVPN on Ubuntu, generate a PKI with Easy-RSA, and configure a working server for IPv4 remote access.

OpenVPN is a mature, widely-supported VPN solution that uses TLS for key exchange and can operate over both UDP and TCP. This guide walks through a complete server setup on Ubuntu.

## Step 1: Install OpenVPN and Easy-RSA

```bash
sudo apt update
sudo apt install openvpn easy-rsa -y
```

## Step 2: Set Up the PKI with Easy-RSA

Easy-RSA is a certificate authority toolkit bundled with OpenVPN.

```bash
# Create and navigate to the Easy-RSA directory

make-cadir ~/openvpn-ca
cd ~/openvpn-ca

# Initialize the PKI
./easyrsa init-pki

# Build the Certificate Authority (press Enter to accept defaults or set a passphrase)
./easyrsa build-ca nopass

# Generate the server certificate and key (no passphrase for automation)
./easyrsa gen-req server nopass
./easyrsa sign-req server server

# Generate Diffie-Hellman parameters (this takes a few minutes)
./easyrsa gen-dh

# Generate a TLS authentication key for extra security
openvpn --genkey secret ~/openvpn-ca/pki/ta.key
```

## Step 3: Copy Certificates to OpenVPN Directory

```bash
sudo cp ~/openvpn-ca/pki/ca.crt /etc/openvpn/server/
sudo cp ~/openvpn-ca/pki/issued/server.crt /etc/openvpn/server/
sudo cp ~/openvpn-ca/pki/private/server.key /etc/openvpn/server/
sudo cp ~/openvpn-ca/pki/dh.pem /etc/openvpn/server/
sudo cp ~/openvpn-ca/pki/ta.key /etc/openvpn/server/
```

## Step 4: Create the Server Configuration

```bash
# Copy the sample server config as a starting point
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server/server.conf
```

Edit `/etc/openvpn/server/server.conf`:

```conf
# Listen on UDP port 1194
port 1194
proto udp

# Use routed tunnel (TUN) for IPv4
dev tun

# Certificate and key paths
ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key
dh /etc/openvpn/server/dh.pem

# IPv4 subnet for VPN clients
server 10.8.0.0 255.255.255.0

# Keep routing table updated
ifconfig-pool-persist /var/log/openvpn/ipp.txt

# Push default route to clients (full tunnel)
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"

# Keepalive and TLS settings
keepalive 10 120
tls-auth /etc/openvpn/server/ta.key 0
cipher AES-256-CBC
user nobody
group nogroup
persist-key
persist-tun

# Logging
status /var/log/openvpn/openvpn-status.log
verb 3
```

## Step 5: Enable Forwarding, NAT, and Start OpenVPN

```bash
# Enable IPv4 forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf

# Add NAT rule
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

# Start and enable OpenVPN
sudo systemctl start openvpn-server@server
sudo systemctl enable openvpn-server@server

# Verify it's running
sudo systemctl status openvpn-server@server
```

Clients can now connect using certificate-based authentication.
