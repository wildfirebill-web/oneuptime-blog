# How to Configure NAT on Linux Using nftables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, Linux, nftables, IPv4

Description: Learn how to configure NAT, MASQUERADE, SNAT, DNAT, and port forwarding on Linux using nftables - the modern replacement for iptables.

## Why nftables?

nftables is the modern Linux packet filtering framework that replaces iptables. Advantages:

- Unified syntax for IPv4, IPv6, and ARP
- Better performance (single pass, JIT compilation)
- Atomic rule updates
- Available in modern Linux distros (Debian 10+, RHEL 8+)

## Check if nftables Is Available

```bash
nft --version

# Enable nftables service

systemctl enable --now nftables
```

## NAT Table Structure in nftables

```text
table inet nat {
    chain prerouting {
        type nat hook prerouting priority -100;
        # DNAT rules here
    }
    chain postrouting {
        type nat hook postrouting priority 100;
        # SNAT/MASQUERADE rules here
    }
}
```

## Outbound NAT: MASQUERADE

```bash
nft add table inet nat
nft add chain inet nat postrouting { type nat hook postrouting priority 100 \; }
nft add rule inet nat postrouting ip saddr 192.168.1.0/24 oifname "eth1" masquerade

# Enable IP forwarding
sysctl -w net.ipv4.ip_forward=1
```

## Outbound NAT: SNAT (Static IP)

```bash
nft add rule inet nat postrouting \
    ip saddr 192.168.1.0/24 oifname "eth1" \
    snat to 203.0.113.1
```

## Port Forwarding: DNAT

```bash
nft add chain inet nat prerouting { type nat hook prerouting priority -100 \; }

# Forward port 80 to 192.168.1.10
nft add rule inet nat prerouting \
    iifname "eth1" tcp dport 80 \
    dnat to 192.168.1.10:80

# Forward port 2222 to SSH on 192.168.1.20:22
nft add rule inet nat prerouting \
    iifname "eth1" tcp dport 2222 \
    dnat to 192.168.1.20:22
```

## Complete nftables NAT Configuration File

```bash
cat > /etc/nftables.conf << 'EOF'
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
    chain forward {
        type filter hook forward priority 0; policy drop;
        ct state related,established accept
        iifname "eth0" oifname "eth1" accept
        iifname "eth1" tcp dport { 80, 443 } ip daddr 192.168.1.10 accept
    }
}

table inet nat {
    chain prerouting {
        type nat hook prerouting priority -100;
        iifname "eth1" tcp dport 80 dnat to 192.168.1.10:80
        iifname "eth1" tcp dport 443 dnat to 192.168.1.10:443
    }
    chain postrouting {
        type nat hook postrouting priority 100;
        ip saddr 192.168.1.0/24 oifname "eth1" masquerade
    }
}
EOF

# Apply
nft -f /etc/nftables.conf
```

## Managing nftables Rules

```bash
# List all rules
nft list ruleset

# List specific table
nft list table inet nat

# Delete a specific rule by handle
nft list ruleset -a   # shows handle numbers
nft delete rule inet nat postrouting handle 4

# Flush all NAT rules
nft flush table inet nat

# Reload configuration
systemctl reload nftables
```

## Viewing Active NAT Connections

```bash
# nftables uses conntrack for state tracking
conntrack -L | grep ESTABLISHED | head -20
conntrack -L | wc -l
```

## Key Takeaways

- nftables uses tables, chains, and rules with a unified syntax.
- `masquerade` automatically uses the current IP of the outgoing interface.
- `snat to IP` sets a fixed source IP; `dnat to IP:port` sets a fixed destination.
- `/etc/nftables.conf` is the persistent configuration file.

**Related Reading:**

- [How to Configure NAT on Linux Using iptables](https://oneuptime.com/blog/post/2026-03-20-nat-linux-iptables/view)
- [How to Configure PAT (Port Address Translation)](https://oneuptime.com/blog/post/2026-03-20-configure-pat-nat-overload/view)
- [How to Configure Source NAT (SNAT) on Linux](https://oneuptime.com/blog/post/2026-03-20-configure-snat-linux/view)
