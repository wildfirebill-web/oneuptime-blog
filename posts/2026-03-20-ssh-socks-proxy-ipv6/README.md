# How to Use SSH as a SOCKS Proxy for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SSH, SOCKS5, IPv6, Dynamic Port Forwarding, Tunnel, Proxy

Description: Use SSH dynamic port forwarding to create an IPv6-capable SOCKS5 proxy, enabling secure tunneling of IPv6 and IPv4 traffic through an SSH server.

## Introduction

SSH's dynamic port forwarding (`-D`) creates a local SOCKS5 proxy that tunnels all connections through the SSH server. The SSH server can use IPv6 to connect to destinations, and the local SOCKS5 proxy accepts connections from IPv6 clients.

## Step 1: Basic SSH SOCKS Proxy

```bash
# Create SOCKS5 proxy on port 1080 via IPv6 SSH server
ssh -D 1080 user@2001:db8::sshserver

# Bind the SOCKS port to localhost IPv6 only
ssh -D "[::1]:1080" user@2001:db8::sshserver

# Background mode with no pseudo-terminal
ssh -D "[::1]:1080" -N -f user@2001:db8::sshserver

# Options:
# -D [bind_address:]port : Dynamic port forwarding (SOCKS)
# -N : Don't execute remote command
# -f : Go to background after authentication
```

## Step 2: SSH Config for Persistent SOCKS Proxy

```
# ~/.ssh/config

Host ipv6-proxy
    HostName 2001:db8::sshserver
    User myuser
    Port 22
    DynamicForward [::1]:1080
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ExitOnForwardFailure yes
    # Use IPv6 only for the SSH connection
    AddressFamily inet6
```

```bash
# Connect using config
ssh -N ipv6-proxy
```

## Step 3: Use the SOCKS Proxy for IPv6

```bash
# curl via SOCKS5 proxy
curl --socks5-hostname "[::1]:1080" http://example.com/
curl --socks5-hostname "[::1]:1080" http://[2001:db8::server]/

# wget via SOCKS proxy
# Export proxy environment variables
export ALL_PROXY="socks5h://[::1]:1080"
wget http://example.com/

# Test IPv6 exit node
curl --socks5-hostname "[::1]:1080" http://ipv6.icanhazip.com/
# Returns the IPv6 address of the SSH server
```

## Step 4: Configure Applications to Use the SOCKS Proxy

```python
# Python — use SOCKS5 via socks module
import socks
import socket

socks.set_default_proxy(socks.SOCKS5, "::1", 1080)
socket.socket = socks.socksocket

import urllib.request
# All requests now go through the SSH SOCKS proxy
response = urllib.request.urlopen("http://[2001:db8::service]/")
print(response.read())
```

```bash
# ProxyChains via SSH SOCKS
# /etc/proxychains.conf
# socks5 ::1 1080

proxychains curl http://[2001:db8::server]/
proxychains ssh user@internal-ipv6-host
```

## Step 5: Autossh for Persistent Proxy

```bash
# Install autossh for automatic reconnection
apt-get install -y autossh

# Run persistent SOCKS proxy with autossh
autossh -M 0 -N \
    -D "[::1]:1080" \
    -o "ServerAliveInterval 30" \
    -o "ServerAliveCountMax 3" \
    user@2001:db8::sshserver &

# Systemd service for persistent proxy
cat > /etc/systemd/system/ssh-socks-proxy.service << 'EOF'
[Unit]
Description=SSH SOCKS5 IPv6 Proxy
After=network.target

[Service]
ExecStart=/usr/bin/autossh -M 0 -N \
    -D "[::1]:1080" \
    -o "ServerAliveInterval 30" \
    user@2001:db8::sshserver
Restart=always
RestartSec=5
User=proxyuser

[Install]
WantedBy=multi-user.target
EOF

systemctl enable --now ssh-socks-proxy
```

## Step 6: Firefox SOCKS Proxy via IPv6

```
Firefox Settings → Proxy → Manual configuration
SOCKS Host: ::1
Port: 1080
SOCKS v5

Check: Proxy DNS when using SOCKS v5
```

## Conclusion

SSH dynamic port forwarding creates a SOCKS5 proxy in seconds: `ssh -D "[::1]:1080" user@2001:db8::server`. The proxy accepts local connections (IPv4 or IPv6) and tunnels them through the SSH server, which can reach IPv6 destinations. Use autossh for persistent proxies and systemd for process management. Monitor the SSH tunnel health with OneUptime by checking the SOCKS5 port availability.
