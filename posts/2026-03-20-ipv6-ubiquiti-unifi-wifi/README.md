# How to Configure IPv6 on Ubiquiti UniFi Wi-Fi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Ubiquiti, UniFi, Wi-Fi, SLAAC, DHCPv6, Wireless LAN

Description: Configure IPv6 support on Ubiquiti UniFi access points and controller including prefix delegation, DHCPv6, SLAAC for wireless clients, and IPv6 firewall policies.

---

Ubiquiti UniFi networks support IPv6 through the UniFi Network Controller. You configure IPv6 at the network level, and UniFi APs automatically forward RA messages and DHCPv6 to wireless clients. The USG or UDM router handles prefix delegation from the ISP.

## UniFi Controller IPv6 Configuration

```
UniFi Controller: Settings > Networks > [Select Network] > IPv6

Options:
- DHCPv6: USG/UDM requests prefix from ISP via DHCPv6-PD
- Static: Manual prefix assignment
- SLAAC only: Router advertisements without DHCPv6
- None: Disable IPv6

Recommended for most deployments:
- IPv6 Interface Type: Prefix Delegation
- Prefix Delegation Size: /64 (per network)
- DHCPv6 Start/Stop: ::2 to ::7d1
```

## UniFi USG IPv6 Configuration via JSON Override

```json
// /usr/lib/unifi/data/sites/default/config.gateway.json
// IPv6 prefix delegation from ISP

{
  "interfaces": {
    "ethernet": {
      "eth0": {
        "description": "WAN",
        "dhcpv6-pd": {
          "pd": {
            "0": {
              "interface": {
                "eth1": {
                  "host-address": "::1",
                  "no-dns": "''"
                }
              },
              "prefix-length": "/48"
            }
          },
          "rapid-commit": "enable"
        }
      },
      "eth1": {
        "description": "LAN",
        "ipv6": {
          "address": {
            "autoconf": "''"
          },
          "dup-addr-detect-transmits": 1,
          "router-advert": {
            "cur-hop-limit": 64,
            "managed-flag": "false",
            "max-interval": 30,
            "name-server": "2001:4860:4860::8888",
            "other-config-flag": "false",
            "prefix": {
              "::/64": {
                "autonomous-flag": "true",
                "on-link-flag": "true",
                "valid-lifetime": 2592000
              }
            },
            "send-advert": "true"
          }
        }
      }
    }
  }
}
```

## UniFi Dream Machine (UDM) IPv6 Setup

```bash
# SSH into UDM
ssh root@192.168.1.1

# Check IPv6 status
ip -6 addr show
ip -6 route show

# Check prefix delegation received from ISP
journalctl -u odhcp6c -f

# Verify IPv6 on LAN bridge
ip -6 addr show br0

# Check RA daemon status
ps aux | grep radvd

# Manually test RA on LAN interface
radvdump -i br0 2>/dev/null | head -30

# View IPv6 DHCP leases
cat /var/lib/misc/dnsmasq.leases | grep -v "^#"
```

## UniFi IPv6 Firewall Rules

```
UniFi Controller > Settings > Firewall & Security > IPv6

Rule 1: Allow Established/Related
- Action: Accept
- IPv6 Protocol: All
- State: Established, Related

Rule 2: Allow ICMPv6
- Action: Accept
- IPv6 Protocol: ICMPv6
- ICMPv6 Type: All

Rule 3: Allow DHCPv6
- Action: Accept
- IPv6 Protocol: UDP
- Destination Port: 546

Rule 4: Block all other inbound
- Action: Drop
- Direction: In
```

```bash
# Via CLI on USG/UDM - view IPv6 firewall rules
show ipv6 firewall name WAN6_LOCAL statistics

# Add rule via VyOS CLI (USG)
configure
set firewall ipv6-name WAN6_LOCAL rule 10 action accept
set firewall ipv6-name WAN6_LOCAL rule 10 protocol icmpv6
commit; save
```

## Verify Wi-Fi Clients Get IPv6

```bash
# On a connected Wi-Fi client
# macOS
networksetup -getinfo Wi-Fi | grep "IPv6"
# Should show:
# IPv6 Address: 2001:db8:1234::/64 derived address

# Windows
netsh interface ipv6 show addresses
# Should show global unicast address

# Linux/Android
ip -6 addr show wlan0

# Test IPv6 connectivity
ping6 2606:4700:4700::1111    # Cloudflare DNS
curl -6 https://ipv6.google.com/  # IPv6 internet

# Speed test via IPv6
curl -6 https://speed.cloudflare.com/cdn-cgi/trace | grep ip
```

## UniFi IPv6 Troubleshooting

```bash
# Check if prefix delegation is working
ssh root@usg-ip
show dhcpv6-pd leases

# Debug RA not reaching clients
tcpdump -i eth1 -nn icmp6 and \(ip6[40]==133 or ip6[40]==134\)
# 133 = Router Solicitation, 134 = Router Advertisement

# Check Wi-Fi association and IPv6 in UniFi logs
grep -i "ipv6\|slaac\|dhcpv6" /var/log/messages

# UniFi AP SSH debug
ssh admin@ap-ip
iwconfig ath0  # Check wireless interface
brctl show     # Verify bridge setup
ip -6 addr show
```

UniFi IPv6 deployment works best with ISP prefix delegation (DHCPv6-PD) configured on the WAN interface, which automatically sub-delegates /64 prefixes to each LAN network. Wireless clients receive IPv6 addresses via SLAAC from the RA broadcast forwarded through the UniFi APs acting as transparent L2 bridges.
