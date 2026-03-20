# How to Understand Inside Local, Inside Global, Outside Local, Outside Global

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, IPv4, Cisco, Concepts

Description: Learn the four NAT address types — Inside Local, Inside Global, Outside Local, and Outside Global — and how they map traffic through a NAT device.

## The Four NAT Address Types

Cisco defines four terms to describe addresses at different points in a NAT translation. Understanding these is essential for configuring and troubleshooting NAT.

| Term | Perspective | Description |
|------|-------------|-------------|
| Inside Local | From inside | Private IP of the internal host |
| Inside Global | From outside | Public IP representing the internal host |
| Outside Local | From inside | IP used to reach the external host (as seen from inside) |
| Outside Global | From outside | Actual IP of the external/destination host |

## Visual Map

```
[Internal Host]          [NAT Router]          [Internet Host]
192.168.1.10     →     203.0.113.1     →     8.8.8.8

Inside Local:  192.168.1.10   ← private IP of your host
Inside Global: 203.0.113.1    ← public IP your host appears as
Outside Local: 8.8.8.8        ← how you see the internet host (usually same as Outside Global)
Outside Global: 8.8.8.8       ← actual IP of the internet host
```

## Standard NAT (Source NAT)

In most home/office NAT configurations:

- **Inside Local** → translated to **Inside Global** (your private IP becomes the public IP)
- **Outside Local = Outside Global** (no translation happens to the destination)

## Destination NAT (DNAT / Port Forwarding)

When hosting a server inside:

```
External Client → 203.0.113.1:80 (Inside Global) → 192.168.1.10:80 (Inside Local)
```

- **Inside Local**: 192.168.1.10 (your web server)
- **Inside Global**: 203.0.113.1 (your public IP)
- **Outside Local** / **Outside Global**: the external client's IP (same, no translation)

## Viewing the Translation Table on Cisco

```cisco
show ip nat translations

! Output columns:
! Pro  Inside global       Inside local        Outside local       Outside global
! tcp  203.0.113.1:1024    192.168.1.10:1024   8.8.8.8:80          8.8.8.8:80
```

## When Outside Local ≠ Outside Global

This happens when **Destination NAT** is applied to outside traffic (less common). Example: you translate 10.0.0.1 → 8.8.8.8 for internal clients:

```
Outside Local:  10.0.0.1  (what inside hosts send to)
Outside Global: 8.8.8.8   (actual destination)
```

## Python: Simulating NAT Translation Table

```python
# Simulate a NAT translation table entry
class NATEntry:
    def __init__(self, inside_local, inside_global, outside_local, outside_global, proto='tcp'):
        self.proto = proto
        self.inside_local = inside_local
        self.inside_global = inside_global
        self.outside_local = outside_local
        self.outside_global = outside_global

    def __repr__(self):
        return (f"{self.proto:5} "
                f"IL={self.inside_local:22} "
                f"IG={self.inside_global:22} "
                f"OL={self.outside_local:22} "
                f"OG={self.outside_global}")

entry = NATEntry('192.168.1.10:54321', '203.0.113.1:1024', '8.8.8.8:80', '8.8.8.8:80')
print(entry)
```

## Quick Reference

| Scenario | Inside Local | Inside Global | Notes |
|----------|-------------|---------------|-------|
| Basic PAT | 192.168.1.x:port | 203.0.113.1:new_port | All shares one public IP |
| Static NAT | 192.168.1.10 | 203.0.113.10 | Fixed 1:1 mapping |
| Port Forward (DNAT) | 192.168.1.10:80 | 203.0.113.1:80 | Inbound only |

## Key Takeaways

- **Inside Local** = private IP of your internal host.
- **Inside Global** = public IP representing your internal host to the outside world.
- **Outside Local** = how inside hosts reach external destinations (often same as Outside Global).
- **Outside Global** = actual IP of the external destination host.

**Related Reading:**

- [How to Configure Static NAT on a Router](https://oneuptime.com/blog/post/2026-03-20-configure-static-nat-router/view)
- [How to Configure PAT (Port Address Translation)](https://oneuptime.com/blog/post/2026-03-20-configure-pat-nat-overload/view)
- [How to Troubleshoot NAT Translation Issues](https://oneuptime.com/blog/post/2026-03-20-troubleshoot-nat-translation/view)
