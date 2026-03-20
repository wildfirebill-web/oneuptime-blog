# How to Understand the TEREDO Address Space (2001::/32)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Teredo, IPv6, Tunneling, NAT Traversal, RFC 4380, Networking

Description: Understand the Teredo IPv6 tunneling mechanism that uses the 2001::/32 address space to provide IPv6 connectivity through IPv4 NAT devices.

## Introduction

Teredo (RFC 4380) is an automatic IPv6 tunneling technology that provides IPv6 connectivity to hosts behind IPv4 NAT devices when no other IPv6 transition mechanism is available. It uses UDP encapsulation to traverse NAT. The `2001::/32` prefix is allocated for Teredo addresses.

## Teredo Address Format

```
2001:0000:SSSS:SSSS:FFFF:UUUU:CCCC:CCCC

Where:
  2001:0000         = Teredo prefix
  SSSS:SSSS         = Teredo server IPv4 address (32 bits)
  FFFF              = Flags (16 bits)
  UUUU              = UDP port (obfuscated, XOR with 0xFFFF)
  CCCC:CCCC         = Client IPv4 address (obfuscated, XOR with 0xFFFFFFFF)
```

## Decoding a Teredo Address

```python
import ipaddress
import socket

def decode_teredo_address(teredo_addr: str) -> dict:
    """
    Decode a Teredo IPv6 address into its components.
    """
    addr = ipaddress.IPv6Address(teredo_addr)
    addr_int = int(addr)

    # Verify it's a Teredo address
    teredo_prefix = ipaddress.IPv6Network("2001::/32")
    if addr not in teredo_prefix:
        raise ValueError(f"{teredo_addr} is not a Teredo address")

    # Extract components (bits 32-63 = server, 64-79 = flags,
    # 80-95 = port, 96-127 = client IP)
    server_ip_int = (addr_int >> 96) & 0xFFFFFFFF
    flags = (addr_int >> 80) & 0xFFFF
    client_port_obfuscated = (addr_int >> 64) & 0xFFFF
    client_ip_obfuscated = addr_int & 0xFFFFFFFF

    # De-obfuscate
    client_port = client_port_obfuscated ^ 0xFFFF
    client_ip_int = client_ip_obfuscated ^ 0xFFFFFFFF

    server_ip = socket.inet_ntoa(server_ip_int.to_bytes(4, 'big'))
    client_ip = socket.inet_ntoa(client_ip_int.to_bytes(4, 'big'))

    return {
        "teredo_addr": teredo_addr,
        "server_ip": server_ip,
        "flags": hex(flags),
        "client_port": client_port,
        "client_nat_ip": client_ip
    }

# Example Teredo address
example = "2001:0:4136:e378:8000:63bf:3fff:fdd2"
try:
    info = decode_teredo_address(example)
    for k, v in info.items():
        print(f"  {k}: {v}")
except Exception as e:
    print(f"Error: {e}")
```

## Security Considerations

Teredo is increasingly deprecated for security reasons:

```bash
# Disable Teredo on Windows (PowerShell)
# netsh interface teredo set state disabled

# Disable Teredo on Linux (it's rarely enabled by default)
# Just ensure no ip6tnl0 or teredo interfaces are up
ip link show | grep teredo

# Block Teredo traffic at the firewall
# Teredo uses UDP port 3544
ip6tables -A INPUT -s 2001::/32 -j LOG --log-prefix "TEREDO-IN: "
iptables -A INPUT -p udp --dport 3544 -j DROP   # Block Teredo server traffic
```

## Modern Alternatives to Teredo

Teredo should be replaced with better mechanisms:

| Mechanism | Use Case | Status |
|---|---|---|
| Teredo | IPv6 behind NAT | Deprecated (security concerns) |
| 6rd | ISP-provided IPv6 | Still used |
| DS-Lite | Dual-stack ISP | Widely deployed |
| MAP-T/MAP-E | IPv6 to IPv4 translation | Growing |
| 464XLAT | Mobile IPv6 | Standard for LTE |

## Detecting Teredo Traffic

```bash
# Identify Teredo packets in tcpdump
sudo tcpdump -i eth0 -n "udp port 3544"

# Identify IPv6 traffic via Teredo tunnel
sudo tcpdump -i eth0 -n "ip6 src 2001::/32"
```

## Conclusion

Teredo's `2001::/32` address space identifies tunnel endpoints and carries NAT traversal metadata. It is largely deprecated due to security issues (Teredo traffic can bypass firewalls). Modern deployments should use 464XLAT or DS-Lite instead. Block Teredo at network boundaries and monitor for unexpected `2001::/32` traffic using OneUptime's network monitoring.
