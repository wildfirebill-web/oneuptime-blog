# How to Migrate from ip6tables to nftables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ip6tables, nftables, Migration, Linux

Description: Step-by-step guide to migrating from ip6tables to nftables, using automated conversion tools and manual review to ensure no security policy gaps during the transition.

## Overview

ip6tables is deprecated on modern Linux distributions in favor of nftables. The `ip6tables-translate` and `iptables-restore-translate` tools can automatically convert most ip6tables rules to nftables syntax. This guide covers the conversion process, common translation issues, and how to verify the migrated rules work correctly.

## Prerequisites

```bash
# Verify nftables is available
nft --version

# Verify translation tools are installed
ip6tables-translate --version
# Part of iptables package on Debian/Ubuntu

# Install if missing
apt install iptables   # Includes translation tools on Debian 10+
```

## Step 1: Export Current ip6tables Rules

```bash
# Save current IPv6 rules to file
ip6tables-save > /tmp/ip6tables-backup.rules

# Verify backup
cat /tmp/ip6tables-backup.rules
```

## Step 2: Translate to nftables Syntax

### Automatic Translation

```bash
# Translate the entire ip6tables ruleset to nftables
iptables-restore-translate -6 -f /tmp/ip6tables-backup.rules > /tmp/nftables-translated.nft

# Review the translated file
cat /tmp/nftables-translated.nft
```

### Translate Individual Rules

```bash
# Translate a single ip6tables rule
ip6tables-translate -A INPUT -p tcp --dport 22 -j ACCEPT
# Output: nft add rule ip6 filter input tcp dport 22 counter accept

ip6tables-translate -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type 134 -j ACCEPT
# Output: nft add rule ip6 filter input ip6 saddr fe80::/10 icmpv6 type nd-router-advert counter accept
```

## Step 3: Review Translation

Common translation issues to check:

### Module-Specific Translations

```bash
# ip6tables -m state → nftables ct state
# ip6tables: -m state --state ESTABLISHED,RELATED -j ACCEPT
# nftables:  ct state established,related accept

# ip6tables -m limit → nftables limit rate
# ip6tables: -m limit --limit 10/s
# nftables:  limit rate 10/second

# ip6tables -m recent → nftables sets (manual conversion required)
# ip6tables: -m recent --name SSH --rcheck --seconds 60 --hitcount 4
# nftables:  meters or sets with timeout (needs manual rewrite)
```

### Things Not Automatically Converted

```bash
# Connection limiting per-IP (recent module)
# ip6tables:
ip6tables -A INPUT -p tcp --dport 22 -m recent --name SSH --rcheck --seconds 60 --hitcount 4 -j DROP
ip6tables -A INPUT -p tcp --dport 22 -m recent --name SSH --set -j ACCEPT

# nftables equivalent (manual):
# table ip6 filter {
#     set SSH_TRACK {
#         type ipv6_addr
#         flags dynamic, timeout
#         timeout 60s
#     }
#     chain input {
#         tcp dport 22 add @SSH_TRACK { ip6 saddr ct count over 4 } drop
#         tcp dport 22 accept
#     }
# }
```

## Step 4: Convert to inet (Unified) Format

After translating ip6tables rules, consider upgrading to unified inet format:

```bash
# ip6 family (translated) → inet family (better)
# Replace:
#   table ip6 filter { ... }
# With:
#   table inet filter { ... }

# Example: Translated ip6-only rule
# nft add rule ip6 filter input tcp dport 22 counter accept

# Unified inet equivalent:
# nft add rule inet filter input tcp dport 22 counter accept
```

## Step 5: Apply and Test

```bash
# 1. Back up current ip6tables rules
ip6tables-save > /tmp/ip6tables-backup.rules

# 2. Safety timer — auto-reverts if testing fails
at now + 5 minutes << 'EOF'
nft flush ruleset
ip6tables-restore < /tmp/ip6tables-backup.rules
EOF

# 3. Stop ip6tables
systemctl stop netfilter-persistent 2>/dev/null
# or: ip6tables -F; ip6tables -P INPUT ACCEPT

# 4. Apply translated nftables rules
nft -f /tmp/nftables-translated.nft

# 5. Test connectivity
ping6 -c 3 2001:4860:4860::8888
ssh user@your-server.com -p 22

# 6. If everything works, cancel the safety timer
atrm $(atq | tail -1 | awk '{print $1}')
```

## Step 6: Disable ip6tables, Enable nftables

```bash
# Disable old ip6tables persistence
systemctl disable netfilter-persistent
systemctl disable ip6tables

# Save new nftables ruleset
nft list ruleset > /etc/nftables.conf

# Enable nftables persistence
systemctl enable --now nftables

# Verify on reboot
systemctl is-enabled nftables
```

## Example: Before and After

### Before (ip6tables)

```bash
ip6tables -P INPUT DROP
ip6tables -A INPUT -i lo -j ACCEPT
ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
ip6tables -A INPUT -p icmpv6 --icmpv6-type packet-too-big -j ACCEPT
ip6tables -A INPUT -s fe80::/10 -p icmpv6 --icmpv6-type neighbour-solicitation -j ACCEPT
ip6tables -A INPUT -p tcp --dport 443 -j ACCEPT
```

### After (nftables)

```bash
table ip6 filter {
    chain input {
        type filter hook input priority 0; policy drop;
        iif "lo" accept
        ct state established,related accept
        icmpv6 type packet-too-big accept
        ip6 saddr fe80::/10 icmpv6 type nd-neighbor-solicit accept
        tcp dport 443 accept
    }
}
```

## Summary

Migrate ip6tables to nftables using `iptables-restore-translate -6 -f rules.file > nftables.nft` for bulk conversion, and `ip6tables-translate` for individual rules. Review the output for module-specific issues (`-m recent` needs manual rewriting). Use a safety timer (`at now + 5 minutes`) that auto-reverts if you lose access. After successful testing, disable `netfilter-persistent` and enable `nftables` with `systemctl enable nftables`. Consider upgrading from `ip6` to `inet` family tables to handle both IPv4 and IPv6 in a unified ruleset.
