# How to Install and Enable iptables on Ubuntu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: iptables, Ubuntu, Linux, Firewall, Security, Installation

Description: Install iptables and iptables-persistent on Ubuntu, understand the default state, and set up a basic working firewall configuration from scratch.

Ubuntu uses nftables as its default firewall backend on modern versions, but iptables remains widely used and available. This guide shows how to install, verify, and start using iptables on Ubuntu.

## Check iptables Status

```bash
# iptables is usually pre-installed on Ubuntu

iptables --version

# View current rules (likely empty on fresh install)
sudo iptables -L -n -v

# Check if iptables or nftables is handling the rules
sudo iptables --version
# "nf_tables" backend → using nftables underneath
# "legacy" → using older iptables backend
```

## Install iptables (if needed)

```bash
# Install iptables
sudo apt update
sudo apt install iptables -y

# Install iptables-persistent (saves rules across reboots)
sudo apt install iptables-persistent -y
# During install, prompted to save current IPv4 and IPv6 rules
```

## Verify Default Chains

```bash
# Check default chain policies (should be ACCEPT on fresh install)
sudo iptables -L INPUT --line-numbers
sudo iptables -L OUTPUT --line-numbers
sudo iptables -L FORWARD --line-numbers

# Three default tables:
# filter (default) - INPUT, FORWARD, OUTPUT chains
# nat             - PREROUTING, POSTROUTING, OUTPUT chains
# mangle          - packet modification
```

## Set Up a Basic Firewall

Start with safe rules that don't lock you out:

```bash
#!/bin/bash
# setup-iptables.sh - Basic firewall setup

# Flush existing rules
sudo iptables -F
sudo iptables -X

# Allow all loopback traffic
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT

# Allow established and related connections
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH from anywhere (change to your IP for security)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP and HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow outbound DNS
sudo iptables -A OUTPUT -p udp --dport 53 -j ACCEPT

# Drop everything else inbound
sudo iptables -A INPUT -j DROP

echo "Firewall rules applied"
sudo iptables -L -n -v
```

## Test Before Applying Default-Deny

NEVER set default policy to DROP before ensuring SSH is allowed:

```bash
# Step 1: Add the allow rules first
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Step 2: Only then set default policy to DROP
sudo iptables -P INPUT DROP

# Step 3: Verify you're still connected
# (If this works, your SSH rule is correct)
echo "Still connected!"
```

## Save and Persist Rules

```bash
# Save rules to persist after reboot
sudo iptables-save > /etc/iptables/rules.v4

# Or use iptables-persistent
sudo netfilter-persistent save

# Verify saved rules
cat /etc/iptables/rules.v4

# Restore manually if needed
sudo iptables-restore < /etc/iptables/rules.v4
```

## Check if Rules Are Active

```bash
# View all chains with packet counts
sudo iptables -L -n -v

# View specific chain
sudo iptables -L INPUT -n -v --line-numbers

# Output includes hit counters:
# pkts bytes target     prot opt in     out     source    destination
#   45  2340 ACCEPT     tcp  --  *      *       0.0.0.0/0 0.0.0.0/0  tcp dpt:22
```

iptables is the foundation of Linux firewalling - even when using UFW or firewalld, those tools generate iptables rules underneath, making iptables knowledge essential.
