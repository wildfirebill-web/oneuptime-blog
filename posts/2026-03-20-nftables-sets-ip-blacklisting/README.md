# How to Use nftables Sets for IP Blacklisting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: nftables, Linux, Firewall, IP Blacklisting, Set, Security, Networking

Description: Build a dynamic IP blacklist using nftables named sets to efficiently block malicious IPs without reloading your entire firewall ruleset.

## Introduction

IP blacklisting allows you to drop traffic from known malicious hosts. With nftables named sets, you can maintain a large blocklist and add or remove entries at runtime without affecting other rules. This is far more scalable than one rule per blocked IP.

## Prerequisites

- nftables installed and running
- Root or sudo access
- Basic familiarity with nftables tables and chains

## Create a Named Blacklist Set

```bash
# Create the filter table if it doesn't exist

nft add table inet filter

# Add an input chain
nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }

# Create a named set for blacklisted IPv4 addresses
nft add set inet filter blacklist { type ipv4_addr \; }

# Drop traffic from any IP in the blacklist (place early in chain for performance)
nft add rule inet filter input ip saddr @blacklist drop
```

## Add IPs to the Blacklist

```bash
# Block a single IP
nft add element inet filter blacklist { 198.51.100.100 }

# Block multiple IPs at once
nft add element inet filter blacklist { 198.51.100.101, 203.0.113.200, 192.0.2.50 }
```

## Blacklist Subnets with CIDR Support

For blocking entire subnets, use the `interval` flag:

```bash
# Create a set that supports CIDR ranges
nft add set inet filter blacklist_subnets { type ipv4_addr \; flags interval \; }

# Add subnets to block
nft add element inet filter blacklist_subnets { 198.51.100.0/24, 203.0.113.0/28 }

# Drop traffic from blacklisted subnets
nft add rule inet filter input ip saddr @blacklist_subnets drop
```

## Full Configuration with Blacklisting

```bash
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    # Blacklisted individual IPs
    set blacklist {
        type ipv4_addr
        elements = { 198.51.100.100, 203.0.113.200 }
    }

    # Blacklisted subnets
    set blacklist_subnets {
        type ipv4_addr
        flags interval
        elements = { 198.51.100.0/24 }
    }

    chain input {
        type filter hook input priority 0; policy drop;

        # Drop blacklisted IPs first (most efficient position)
        ip saddr @blacklist drop
        ip saddr @blacklist_subnets drop

        # Then process legitimate traffic
        iif lo accept
        ct state established,related accept
        ct state invalid drop

        tcp dport 22 accept
        tcp dport { 80, 443 } accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

## Automate Blacklist Updates with a Script

You can automate adding IPs from a threat feed:

```bash
#!/bin/bash
# Script: update-blacklist.sh
# Reads a list of IPs from a file and adds them to nftables blacklist

BLACKLIST_FILE="/etc/nftables-blacklist.txt"
SET_NAME="blacklist"
TABLE="inet filter"

# Read IPs line by line and add to nftables set
while IFS= read -r ip; do
    # Skip empty lines and comments
    [[ -z "$ip" || "$ip" == \#* ]] && continue
    nft add element $TABLE $SET_NAME { "$ip" } 2>/dev/null
done < "$BLACKLIST_FILE"

echo "Blacklist updated: $(nft list set $TABLE $SET_NAME | grep -c 'elements') entries"
```

## Remove an IP from the Blacklist

```bash
# Remove a single IP
nft delete element inet filter blacklist { 198.51.100.100 }

# List current blacklist
nft list set inet filter blacklist
```

## Conclusion

nftables named sets provide an efficient IP blacklisting mechanism that scales to thousands of entries with no performance degradation. Place the drop rules at the top of the input chain for maximum efficiency, and use CIDR-capable sets with the `interval` flag to block entire subnets. Runtime updates mean zero downtime when adding or removing blocked addresses.
