# How to Configure Dante SOCKS5 Proxy for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dante, SOCKS5, IPv6, Proxy, Sockd, Authentication, Dual-Stack

Description: Configure Dante SOCKS5 proxy server to listen on IPv6 addresses, authenticate clients, and forward connections to IPv6 destinations.

## Introduction

Dante is a SOCKS5 server (socksify) and client library. It supports SOCKS4, SOCKS4a, and SOCKS5 protocols with IPv6. Its `sockd` server can listen on IPv6 and route SOCKS connections to IPv6 destinations.

## Installation

```bash
apt-get install -y dante-server
```

## Step 1: sockd.conf for IPv6

```nginx
# /etc/danted.conf

# Log errors and connections

logoutput: syslog

# Internal interface (where clients connect - IPv6)
internal: :: port = 1080

# External interface (where outbound connections go)
external: eth0  # Or specify IPv6 address

# Allow SOCKS5 (method 6 = username/password)
socksmethod: username none

# Client access rules
client pass {
    from: 2001:db8::/32 port 1-65535 to: 0.0.0.0/0
    log: connect disconnect
}

client pass {
    from: ::1/128 port 1-65535 to: 0.0.0.0/0
    log: connect disconnect
}

# Reject all other clients
client block {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: connect
}

# Allow connections to IPv6 destinations
socks pass {
    from: 0.0.0.0/0 to: ::/0
    log: connect disconnect
    socksmethod: username none
}

# Block all others
socks block {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: connect error
}
```

## Step 2: Authentication

```nginx
# /etc/danted.conf

# Require username/password for SOCKS5
socksmethod: username

client pass {
    from: ::/0 to: ::/0
    log: connect
}

socks pass {
    from: 0.0.0.0/0 to: ::/0
    socksmethod: username
    log: connect
}
```

```bash
# Add system user for SOCKS authentication
useradd -r -s /sbin/nologin socksuser
passwd socksuser
```

## Step 3: Start and Test

```bash
# Validate config
danted -V -f /etc/danted.conf

# Start the service
systemctl enable --now danted

# Verify listening on IPv6
ss -lntp | grep :1080

# Test SOCKS5 over IPv6 (no auth)
curl --socks5 "[::1]:1080" http://example.com/

# Test SOCKS5 with auth
curl --socks5 "socksuser:password@[::1]:1080" http://example.com/

# Test connecting to IPv6 destination
curl --socks5 "[::1]:1080" http://[2001:db8::server]/
```

## Step 4: IPv6 Outbound Only

```nginx
# /etc/danted.conf - force outbound via IPv6 interface

external: 2001:db8::proxy  # Specific IPv6 source address

socks pass {
    from: 0.0.0.0/0 to: ::/0
    socksmethod: none
    command: connect
}
```

## Step 5: Test with Python

```python
import socks
import socket

# Configure SOCKS5 proxy over IPv6
socks.set_default_proxy(
    socks.SOCKS5,
    "::1",    # IPv6 proxy address
    1080,
    username="socksuser",
    password="password"
)
socket.socket = socks.socksocket

# Now all socket connections go through the proxy
import urllib.request
response = urllib.request.urlopen("http://ipv6.example.com/")
print(response.read())
```

## Conclusion

Dante's `sockd` listens on IPv6 by setting `internal: :: port = 1080`. Client ACLs accept IPv6 CIDR notation. Both IPv6-to-IPv6 and IPv6-to-IPv4 proxy connections are supported. Use username authentication for security. Monitor Dante with OneUptime's TCP port checks on the SOCKS5 port.
