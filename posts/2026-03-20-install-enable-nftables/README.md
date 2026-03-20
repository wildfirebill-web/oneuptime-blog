# How to Install and Enable nftables on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: nftables, Linux, Firewall, Security, Installation, Networking

Description: Install nftables on Linux, understand how it replaces iptables, and set up the nftables service with a basic ruleset to get started with modern Linux firewalling.

nftables is the modern replacement for iptables, ip6tables, arptables, and ebtables. It unifies all protocol filtering in a single tool with better performance, atomic rule updates, and cleaner syntax.

## Install nftables

```bash
# Debian/Ubuntu
sudo apt install nftables -y

# RHEL/CentOS 7
sudo yum install nftables -y

# RHEL/CentOS 8/9
sudo dnf install nftables -y

# Verify version
nft --version
# nftables v0.9.x (Fenrir)
```

## Enable the nftables Service

```bash
# Enable nftables to start at boot
sudo systemctl enable nftables

# Start the service
sudo systemctl start nftables

# Check status
sudo systemctl status nftables
```

## Check Current Rules

```bash
# List all nftables rules
sudo nft list ruleset

# If empty (fresh install), output will be:
# (empty)

# Check if iptables is also active
sudo iptables -L
# On some systems, iptables rules show nf_tables backend
```

## nftables vs iptables Conceptual Mapping

```
iptables Concept       nftables Equivalent
--------------------   -----------------------------------------
Table (filter, nat)    table inet/ip/ip6 <name>
Chain (INPUT, etc.)    chain <name> { type filter hook input ... }
Rule (-A INPUT ...)    add rule <table> <chain> <expression>
ipset                  nftables sets (built-in, no separate tool)
Multiple tables        Unified: inet handles both IPv4 and IPv6
```

## Basic nftables Configuration

```bash
# Create a basic config file
sudo tee /etc/nftables.conf << 'EOF'
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # Allow loopback
        iif lo accept

        # Allow established and related connections
        ct state established,related accept

        # Drop invalid connections
        ct state invalid drop

        # Allow ICMP (ping)
        icmp type echo-request accept

        # Allow SSH
        tcp dport 22 accept

        # Allow HTTP and HTTPS
        tcp dport { 80, 443 } accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
EOF

# Apply the config
sudo nft -f /etc/nftables.conf

# Verify rules loaded
sudo nft list ruleset
```

## Test Before Enabling at Boot

```bash
# Test syntax without applying
sudo nft -c -f /etc/nftables.conf
# -c = check only (dry run)

# Apply manually and verify connectivity
sudo nft -f /etc/nftables.conf
ping -c 1 8.8.8.8    # Test outbound
ssh user@server       # Test SSH (from another window)

# If everything works, save and enable
sudo systemctl enable nftables
sudo systemctl start nftables
```

## Relationship with iptables

```bash
# On modern kernels, both iptables and nftables use nf_tables
# Don't use both simultaneously — pick one

# Check which tool manages current rules
sudo iptables -L | head -5
sudo nft list ruleset

# If migrating from iptables, translate existing rules first:
iptables-translate -A INPUT -p tcp --dport 22 -j ACCEPT
# Output: nft add rule ip filter INPUT tcp dport 22 accept
```

nftables is the future of Linux packet filtering — it's already the default on Debian 10+, RHEL 8+, and Ubuntu 20.04+, making it the right tool to learn for any new deployment.
