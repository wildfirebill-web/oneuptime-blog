# How to Connect Portainer to a Remote Podman Socket

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Podman, Remote, SSH, Containers

Description: Step-by-step guide to connecting Portainer to a Podman socket running on a remote host using SSH tunneling or TCP socket exposure.

## Introduction

While Portainer can manage a local Podman instance, many production scenarios require connecting Portainer to Podman on a remote host. This guide covers two methods: SSH tunneling and exposing Podman's TCP socket over TLS.

## Prerequisites

- Portainer instance running (local or remote)
- Remote Linux host with Podman installed
- SSH access to the remote host
- (Optional) TLS certificates for secure TCP

## Method 1: SSH Tunneling

SSH tunneling is the safest approach as it does not expose the Podman socket directly.

### On the Remote Host

```bash
# Enable Podman socket on the remote host
systemctl --user enable --now podman.socket

# Check the socket is active
systemctl --user status podman.socket

# Note the socket path
ls -la /run/user/$(id -u)/podman/podman.sock
```

### Setting Up the SSH Tunnel

```bash
# On your Portainer host, create an SSH tunnel
# This forwards local port 9999 to the remote Podman socket
ssh -L 9999:/run/user/1000/podman/podman.sock \
    user@remote-host \
    -N &

# Verify the tunnel is working
curl --unix-socket /tmp/podman-tunnel.sock http://localhost/version
```

For a persistent tunnel, use autossh:

```bash
# Install autossh
sudo apt-get install -y autossh

# Create a persistent tunnel
autossh -M 0 -f \
  -L 9999:/run/user/1000/podman/podman.sock \
  -o "ServerAliveInterval 30" \
  -o "ServerAliveCountMax 3" \
  user@remote-host \
  -N
```

### Configuring Portainer

In Portainer, add the environment using the local forwarded socket:

```
unix:///tmp/podman.sock
```

Or via TCP using the forwarded port:

```
tcp://localhost:9999
```

## Method 2: Exposing Podman TCP Socket

### Generate TLS Certificates

```bash
# Create a directory for certs
mkdir -p ~/podman-tls && cd ~/podman-tls

# Generate CA key and certificate
openssl genrsa -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem \
  -out ca.pem -subj "/CN=podman-ca"

# Generate server key and certificate
openssl genrsa -out server-key.pem 4096
openssl req -new -key server-key.pem \
  -out server.csr -subj "/CN=remote-host"
openssl x509 -req -days 365 -in server.csr \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -out server-cert.pem

# Generate client key and certificate
openssl genrsa -out key.pem 4096
openssl req -new -key key.pem \
  -out client.csr -subj "/CN=client"
openssl x509 -req -days 365 -in client.csr \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -out cert.pem
```

### Configure Podman to Listen on TCP

```bash
# Edit or create the Podman service configuration
sudo mkdir -p /etc/systemd/system/podman.service.d/

# Create a drop-in configuration
sudo tee /etc/systemd/system/podman.service.d/tcp.conf << 'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/podman system service \
  --time=0 \
  tcp://0.0.0.0:2376 \
  --tls-verify \
  --tlscacert=/etc/podman/tls/ca.pem \
  --tlscert=/etc/podman/tls/server-cert.pem \
  --tlskey=/etc/podman/tls/server-key.pem
EOF

sudo systemctl daemon-reload
sudo systemctl restart podman.service
```

### Connecting Portainer via TLS

In Portainer's "Add Environment" dialog:

```
Endpoint URL: tcp://remote-host:2376
TLS: Enabled
CA Certificate: ca.pem
Certificate: cert.pem
Private Key: key.pem
```

## Method 3: Podman via varlink/REST API with Docker-compat

```bash
# Start Podman in service mode (Docker-compatible REST API)
podman system service \
  --time=0 \
  unix:///tmp/podman.sock &

# Test the API
curl -s --unix-socket /tmp/podman.sock \
  http://localhost/v4.0.0/libpod/info | python3 -m json.tool
```

## Verifying the Connection

After configuring in Portainer:

```bash
# From Portainer host, test the connection
curl -s \
  --cert cert.pem \
  --key key.pem \
  --cacert ca.pem \
  https://remote-host:2376/version | python3 -m json.tool
```

## Conclusion

Connecting Portainer to a remote Podman socket enables centralized management of multiple Podman hosts from a single Portainer instance. SSH tunneling is recommended for simplicity and security, while TLS-enabled TCP sockets offer better performance for production environments with many remote nodes.
