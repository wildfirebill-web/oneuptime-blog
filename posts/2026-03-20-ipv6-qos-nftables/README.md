# How to Configure IPv6 QoS with nftables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: nftables, IPv6, QoS, DSCP, Linux, Traffic Marking, Firewall

Description: Configure IPv6 Quality of Service using nftables for DSCP marking and traffic classification, leveraging nftables' native IPv6 support for efficient packet marking.

---

nftables is the modern Linux firewall/packet filtering framework with native IPv6 support. Its `ip6 dscp` match and set operations provide clean syntax for IPv6 DSCP marking compared to the older iptables/ip6tables approach.

## nftables IPv6 DSCP Marking

```bash
# /etc/nftables.conf - IPv6 QoS with DSCP marking

flush ruleset

table ip6 mangle {

    chain prerouting {
        type filter hook prerouting priority mangle; policy accept;

        # Mark VoIP RTP media - EF (DSCP 46)
        meta l4proto udp udp dport 10000-20000 ip6 dscp set ef

        # Mark SIP signaling - CS5 (DSCP 40)
        meta l4proto { tcp, udp } th dport 5060 ip6 dscp set cs5

        # Mark video streaming - AF41 (DSCP 34)
        meta l4proto tcp tcp dport { 1935, 8554 } ip6 dscp set af41

        # Mark interactive SSH - AF31 (DSCP 26)
        meta l4proto tcp tcp dport 22 ip6 dscp set af31

        # Mark DNS - CS6 (DSCP 48) - network control
        meta l4proto { tcp, udp } th dport 53 ip6 dscp set cs6

        # Mark bulk transfers - AF11 (DSCP 10)
        meta l4proto tcp tcp dport 8080-8090 ip6 dscp set af11

        # Default - best effort CS0 for unmatched
        ip6 dscp != { ef, cs5, af41, af31, cs6, af11 } ip6 dscp set cs0
    }

    chain postrouting {
        type filter hook postrouting priority mangle; policy accept;

        # Trust EF/CS5 from internal networks
        # Remark all from external to default
        ip6 saddr != 2001:db8:internal::/48 ip6 dscp set cs0
    }

}
```

```bash
# Apply nftables configuration

sudo nft -f /etc/nftables.conf

# Verify rules loaded
sudo nft list table ip6 mangle

# Enable nftables on boot
sudo systemctl enable nftables
sudo systemctl start nftables
```

## Source-Based DSCP Marking

```bash
# /etc/nftables.conf - Source-based IPv6 marking

table ip6 mangle {

    chain prerouting {
        type filter hook prerouting priority mangle; policy accept;

        # Mark traffic from VoIP phones subnet
        ip6 saddr 2001:db8:voip::/64 ip6 dscp set ef

        # Mark traffic from video conferencing systems
        ip6 saddr 2001:db8:video::/64 ip6 dscp set af41

        # Mark traffic from server farm (low priority outbound)
        ip6 saddr 2001:db8:servers::/48 meta l4proto tcp \
          tcp dport != 443 ip6 dscp set af21

        # Mark ICMP6 traffic (essential - network control)
        ip6 nexthdr icmpv6 ip6 dscp set cs7
    }

}
```

## Combining DSCP and Conntrack for IPv6

```bash
# Mark connections and track state with nftables

table ip6 mangle {

    chain prerouting {
        type filter hook prerouting priority mangle; policy accept;

        # Mark VoIP connection and all related packets
        meta l4proto udp udp dport 5060 ct state new \
          ip6 dscp set cs5 \
          ct mark set 1  # Mark connection

        # Apply marking to all packets in VoIP connections
        ct mark 1 ip6 dscp set ef

        # Video connections
        meta l4proto tcp tcp dport 443 ct state new \
          ct mark set 2

        ct mark 2 ip6 dscp set af41
    }

}
```

## nftables Priority Map for IPv6

```bash
# Use nftables maps for efficient DSCP lookup

table ip6 mangle {

    # Map destination ports to DSCP values
    map port_dscp {
        type inet_service : dscp;
        elements = {
            5060 : cs5,
            5061 : cs5,
            22   : af31,
            80   : cs0,
            443  : cs0,
            8080 : af11,
            25   : cs0,
        }
    }

    chain prerouting {
        type filter hook prerouting priority mangle; policy accept;

        # Apply DSCP from port map (TCP)
        meta l4proto tcp ip6 dscp set tcp dport map @port_dscp

        # Apply DSCP from port map (UDP)
        meta l4proto udp ip6 dscp set udp dport map @port_dscp

        # VoIP media by port range
        meta l4proto udp udp dport 10000-20000 ip6 dscp set ef
    }

}
```

## Monitoring and Debugging nftables IPv6 QoS

```bash
# Check current DSCP marking rules
sudo nft list table ip6 mangle

# Add counters to rules for monitoring
sudo nft add counter ip6 mangle voip_counter

# Update rule to use counter
sudo nft replace rule ip6 mangle prerouting \
  meta l4proto udp udp dport 10000-20000 \
  counter name voip_counter \
  ip6 dscp set ef

# Check counter
sudo nft list counters | grep "voip"

# Trace packets for debugging
sudo nft add table ip6 trace
sudo nft add chain ip6 trace output { type filter hook output priority 0; }
sudo nft add rule ip6 trace output \
  ip6 saddr 2001:db8::client \
  meta nftrace set 1
sudo nft monitor trace | head -50

# Cleanup trace when done
sudo nft delete table ip6 trace
```

nftables provides cleaner IPv6 DSCP syntax with `ip6 dscp set ef` compared to ip6tables, and its map and set constructs enable efficient port-to-DSCP lookups without long chains of individual rules, making it the preferred approach for modern Linux IPv6 QoS configuration.
