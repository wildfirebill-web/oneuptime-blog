# How to Migrate Firewall Rules from iptables to nftables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: nftables, iptables, Linux, Firewall, Migration, IPv4

Description: Migrate existing iptables IPv4 firewall rules to nftables using automated translation tools and manual verification steps.

nftables is the modern Linux firewall framework that replaces iptables. It has a cleaner syntax, better performance for large rulesets, and combines IPv4/IPv6 in unified rules.

## Why Migrate to nftables?

- Unified framework for IPv4 and IPv6
- Better performance with large rulesets
- Cleaner, more readable syntax
- Atomic rule updates
- Available in all modern Linux distributions

## Step 1: View Current iptables Rules

```bash
# Save current iptables rules
sudo iptables-save > /tmp/iptables-backup.txt
cat /tmp/iptables-backup.txt
```

## Step 2: Automated Translation with iptables-translate

```bash
# Translate individual rules
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
# → nft add rule ip filter input tcp dport 22 counter accept

# Use iptables-translate for each rule
iptables-translate -A INPUT -s 192.168.1.0/24 -p tcp --dport 443 -j ACCEPT
# Output: nft add rule ip filter input ip saddr 192.168.1.0/24 tcp dport 443 counter accept
```

## Step 3: Bulk Translation with iptables-restore-translate

```bash
# Translate the entire saved iptables config at once
sudo iptables-save | iptables-restore-translate -f > /tmp/nftables-translated.nft

# Review the translation
cat /tmp/nftables-translated.nft
```

## Example Translation

iptables rules:

```bash
# Original iptables rules
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp --dport 22 -j ACCEPT
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT
COMMIT
```

Translated nftables rules:

```
table ip filter {
    chain INPUT {
        type filter hook input priority 0; policy drop;
        ct state established,related counter accept
        iifname "lo" counter accept
        tcp dport 22 counter accept
        tcp dport 80 counter accept
        tcp dport 443 counter accept
    }

    chain FORWARD {
        type filter hook forward priority 0; policy drop;
    }

    chain OUTPUT {
        type filter hook output priority 0; policy accept;
    }
}
```

## Step 4: Apply nftables Rules

```bash
# Apply translated rules
sudo nft -f /tmp/nftables-translated.nft

# Verify rules are loaded
sudo nft list ruleset

# Test connectivity before making permanent
ssh <remote-host>  # Test SSH still works
curl http://localhost  # Test HTTP if applicable
```

## Step 5: Disable iptables and Enable nftables

```bash
# On Ubuntu 20.04+, switch to nftables
sudo update-alternatives --set iptables /usr/sbin/iptables-nft
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-nft

# Ensure nftables service is enabled
sudo systemctl enable nftables
sudo systemctl start nftables

# Save nftables rules
sudo cp /tmp/nftables-translated.nft /etc/nftables.conf
```

## Step 6: Verify the Migration

```bash
# Confirm nftables rules are active
sudo nft list ruleset

# Confirm iptables legacy is not running
sudo iptables-legacy -L 2>/dev/null || echo "Legacy iptables not active"

# Test all previously open services are still accessible
nc -zv localhost 22
nc -zv localhost 80
nc -zv localhost 443
```

The automated `iptables-restore-translate` handles most standard rules correctly, but review the output for complex matches like `--state`, custom chains, and NAT rules which may need manual adjustment.
