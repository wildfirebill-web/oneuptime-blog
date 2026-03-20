# How Teredo Tunneling Provides IPv6 Connectivity Through NAT

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Teredo, IPv6, NAT, Tunneling, IPv4, Transition

Description: Understand how Teredo tunneling encapsulates IPv6 packets in IPv4 UDP to traverse NAT devices, configure Teredo on Linux with miredo, and understand its use cases and limitations.

## Introduction

Teredo (RFC 4380) is a last-resort IPv6 tunneling mechanism that works even through NAT. It encapsulates IPv6 packets in IPv4 UDP (port 3544), allowing NAT traversal. Teredo addresses use the 2001::/32 prefix. While it works through NAT, it is deprecated and slow compared to native IPv6 or 6in4.

## Teredo Address Format

```text
Teredo address: 2001:0000:server_ip:flags:obscured_client_port:obscured_client_ip

Example:
2001:0:4137:9e74:8c1:8ff:fe80:ffa2
         ↑ Teredo server IPv4
                   ↑ Flags
                          ↑ Obscured UDP port (XOR with 0xffff)
                                 ↑ Obscured client IPv4 (XOR)
```

## Installing Miredo (Linux Teredo Client)

```bash
# Debian/Ubuntu

sudo apt install miredo

# RHEL/CentOS
sudo dnf install miredo

# Start miredo
sudo systemctl enable --now miredo

# Verify Teredo interface
ip address show teredo
# Shows: 2001:... address
```

## Miredo Configuration

```bash
# /etc/miredo/miredo.conf

# Teredo server (use a public Teredo server)
ServerAddress teredo.remlab.net
# or:
# ServerAddress teredo2.remlab.net

# Local port (default 3544 - can change to avoid blocks)
BindPort 3544

# Bind to specific IPv4 (optional)
# BindAddress 10.0.0.5

# Interface name
InterfaceName teredo
```

## Testing Teredo Connectivity

```bash
# Check Teredo interface after starting miredo
ip address show teredo
ip -6 route show | grep teredo

# Test IPv6 connectivity via Teredo
ping -6 2001:4860:4860::8888
curl -6 https://ipv6.google.com

# Check Teredo is working
miredo-checkconf /etc/miredo/miredo.conf
```

## Teredo vs Other Tunneling Methods

| Feature | Teredo | 6in4 (HE) | 6to4 |
|---|---|---|---|
| Works through NAT | Yes | No | No |
| Protocol | UDP 3544 | IP proto 41 | IP proto 41 |
| Address space | 2001::/32 | Provider-assigned | 2002::/16 |
| Reliability | Low | High | Medium |
| Performance | Slowest | Best | Medium |
| Registration required | No | Yes (free) | No |
| Status | Deprecated | Active | Deprecated |

## Windows Teredo

```powershell
# Check Teredo status on Windows
netsh interface teredo show state

# Enable Teredo
netsh interface teredo set state type=default

# Set Teredo server
netsh interface teredo set state servername=teredo.remlab.net

# Disable Teredo (if native IPv6 is available)
netsh interface teredo set state disabled
```

## Security Considerations

```bash
# Teredo can bypass IPv6 firewalls if only IPv4 is filtered
# Check if Teredo is creating unexpected IPv6 tunnels:
ip address show | grep "2001:"

# Block Teredo on corporate networks if not needed:
# Block UDP port 3544 inbound/outbound
sudo iptables -A OUTPUT -p udp --dport 3544 -j DROP
sudo iptables -A INPUT  -p udp --sport 3544 -j DROP
```

## Conclusion

Teredo is a last-resort tunneling mechanism for IPv6 through NAT. Install `miredo` on Linux for Teredo support. For reliable IPv6 connectivity, prefer Hurricane Electric's 6in4 tunnels (tunnelbroker.net) which require a public IPv4 but offer far better performance and stability. Teredo is deprecated and should only be used when no better option is available. On corporate networks, block UDP 3544 to prevent unauthorized Teredo tunnels.
