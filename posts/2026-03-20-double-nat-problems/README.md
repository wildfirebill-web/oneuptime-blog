# How to Diagnose and Fix Double NAT Problems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, IPv4, Troubleshooting

Description: Learn what double NAT is, how it affects connectivity, how to detect it, and the best ways to fix it.

## What Is Double NAT?

Double NAT occurs when traffic passes through two NAT devices before reaching the internet. This is common when:

- Your ISP provides a modem/router with NAT
- You add your own router behind it
- Both devices perform NAT

```text
[Your PC]         [Your Router]          [ISP Modem/Router]         [Internet]
192.168.1.10 → NAT → 192.168.0.100 → NAT → 203.0.113.1 → 8.8.8.8
```

## Problems Caused by Double NAT

1. **Port forwarding fails** - forwarding on your router goes to its NAT'd IP, not the end host
2. **VoIP/SIP issues** - SIP ALG interacts badly with nested NAT
3. **VPN problems** - IPsec/IKE NAT traversal is complicated by double NAT
4. **Gaming issues** - strict NAT type (Type 3 on PlayStation, NAT D on others)
5. **UPnP doesn't work** - UPnP only programs one router's port forwards

## Detecting Double NAT

```bash
# Check your private IP

ip addr show

# Check the next hop (gateway)
ip route show default | awk '/default/ {print $3}'

# Trace the path (if second hop is also private, you have double NAT)
traceroute 8.8.8.8

# Example output showing double NAT:
# 1. 192.168.1.1  (your router)
# 2. 192.168.0.1  (ISP modem/router)
# 3. 203.0.113.x  (ISP public IP)
```

If hop 2 is a private IP, you are behind double NAT.

```bash
# Python: Detect double NAT by checking external IP vs gateway
import subprocess, urllib.request

gateway = subprocess.run(
    ['ip', 'route', 'show', 'default'],
    capture_output=True, text=True
).stdout.split()[2]

external_ip = urllib.request.urlopen('https://ifconfig.me').read().decode()

print(f"Gateway: {gateway}")
print(f"External IP: {external_ip}")
print(f"Double NAT suspected: gateway is private, external IP is different")
```

## Fix 1: Enable Bridge/Passthrough Mode on ISP Modem

Most ISP modems support a "bridge mode" that disables their NAT and acts as a pure modem:

1. Log into ISP modem (usually 192.168.0.1 or 192.168.100.1)
2. Find WAN or Internet settings
3. Enable **Bridge Mode** or **IP Passthrough**
4. Your router now gets the public IP directly

## Fix 2: DMZ Host on ISP Modem

If bridge mode isn't available, put your router in the ISP modem's DMZ:

1. Log into ISP modem
2. Find **DMZ** or **Exposed Host** settings
3. Set DMZ host to your router's WAN IP (e.g., 192.168.0.100)
4. Now all inbound traffic goes to your router

## Fix 3: Use ISP Modem as Your Only Router

If your ISP modem has enough features (firewall, VPN, etc.), remove your own router and connect devices directly to the ISP modem.

## Fix 4: Accept Double NAT (Workarounds)

If you can't eliminate double NAT:

```bash
# Forward ports on BOTH routers
# ISP modem: Forward port 80 → your router's WAN IP (192.168.0.100)
# Your router: Forward port 80 → internal server (192.168.1.10)
```

This works but requires configuring both devices.

## Verifying Double NAT Is Fixed

```bash
# After fix: traceroute should show public IP at hop 2
traceroute 8.8.8.8
# 1. 192.168.1.1  (your router)
# 2. 203.0.113.1  (ISP public IP - no more intermediate private hop)
```

## Key Takeaways

- Double NAT occurs when two NAT devices are in series on the path to the internet.
- Detection: check if `traceroute` shows private IPs at two consecutive hops.
- Best fix: enable bridge/passthrough mode on the ISP modem.
- Second best: put your router in the ISP modem's DMZ.

**Related Reading:**

- [How to Configure Hairpin NAT for Internal Access to Public Services](https://oneuptime.com/blog/post/2026-03-20-hairpin-nat-internal-access/view)
- [How to Detect If You Are Behind Carrier-Grade NAT (CGNAT)](https://oneuptime.com/blog/post/2026-03-20-detect-cgnat/view)
- [How to Troubleshoot NAT Translation Issues](https://oneuptime.com/blog/post/2026-03-20-troubleshoot-nat-translation/view)
