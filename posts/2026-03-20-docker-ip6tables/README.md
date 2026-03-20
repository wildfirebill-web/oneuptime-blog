# How to Enable ip6tables in Docker for IPv6 Network Isolation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, ip6tables, Firewall, Network Isolation, Security

Description: Enable and configure ip6tables in Docker to provide IPv6 network isolation between containers, manage IPv6 firewall rules for Docker networks, and understand how Docker manages ip6tables rules automatically.

## Introduction

When `ip6tables` is enabled in Docker's `daemon.json`, Docker automatically manages `ip6tables` rules to provide network isolation for IPv6 containers. Without `ip6tables`, all containers on all Docker networks can reach each other over IPv6, defeating network isolation. Docker creates chains like `DOCKER` and `DOCKER-ISOLATION-STAGE-1` in ip6tables to enforce per-network isolation rules.

## Enable ip6tables in daemon.json

```json
// /etc/docker/daemon.json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:dead:beef::/80",
  "ip6tables": true
}
```

```bash
sudo systemctl restart docker

# Verify ip6tables rules are being managed by Docker
sudo ip6tables -L DOCKER -n
# Should show DOCKER chain with per-network rules

sudo ip6tables -L FORWARD -n
# Should show DOCKER-USER and DOCKER-ISOLATION chains
```

## View Docker-Managed ip6tables Rules

```bash
# List all ip6tables rules in all tables
sudo ip6tables -L -n -v

# List Docker-specific chains
sudo ip6tables -L DOCKER -n -v
sudo ip6tables -L DOCKER-ISOLATION-STAGE-1 -n -v
sudo ip6tables -L DOCKER-ISOLATION-STAGE-2 -n -v

# Nat table (for masquerading IPv6 outbound traffic)
sudo ip6tables -t nat -L -n -v

# Docker IPv6 MASQUERADE rule
sudo ip6tables -t nat -L POSTROUTING -n -v
# Shows: MASQUERADE for container IPv6 prefix
```

## Add Custom ip6tables Rules

```bash
# Docker creates a DOCKER-USER chain for custom rules
# Always add custom rules to DOCKER-USER so they persist

# Block inbound IPv6 from specific range
sudo ip6tables -I DOCKER-USER -s 2001:db8:blocked::/48 -j DROP

# Allow only specific IPv6 prefix to reach containers
sudo ip6tables -I DOCKER-USER -s 2001:db8:trusted::/48 -j ACCEPT
sudo ip6tables -A DOCKER-USER -s "::/0" -j DROP

# Rate limit IPv6 connections
sudo ip6tables -I DOCKER-USER -m limit --limit 100/s --limit-burst 200 -j ACCEPT
sudo ip6tables -A DOCKER-USER -j DROP

# View DOCKER-USER rules
sudo ip6tables -L DOCKER-USER -n -v
```

## Persist ip6tables Rules Across Reboots

```bash
# Install iptables-persistent
sudo apt-get install iptables-persistent

# Save current rules (includes ip6tables)
sudo netfilter-persistent save

# Rules saved to:
# /etc/iptables/rules.v4
# /etc/iptables/rules.v6

# Verify saved ip6tables rules
cat /etc/iptables/rules.v6

# Restore manually if needed
sudo ip6tables-restore < /etc/iptables/rules.v6
```

## Troubleshoot ip6tables Issues

```bash
# Problem: IPv6 container traffic not isolated between networks
# Check: ip6tables is not enabled
docker info | grep -i "ip6tables"

# If not shown, enable in daemon.json and restart

# Problem: Container cannot reach IPv6 internet
# Check FORWARD chain default policy
sudo ip6tables -L FORWARD --policy

# If DROP, Docker should add ACCEPT rules — verify Docker chain
sudo ip6tables -L FORWARD -n | grep DOCKER

# Check if MASQUERADE is in place for outbound
sudo ip6tables -t nat -L POSTROUTING -n | grep fd00

# Problem: Custom rules disappear after Docker restart
# Always use DOCKER-USER chain, not DOCKER chain
# Docker flushes and re-creates DOCKER chain on restart
# DOCKER-USER is preserved
```

## Conclusion

Enable `ip6tables` in Docker's `daemon.json` to get automatic IPv6 network isolation between Docker networks. Docker manages `DOCKER`, `DOCKER-ISOLATION-STAGE-1`, and `DOCKER-ISOLATION-STAGE-2` chains automatically. Add custom rules to the `DOCKER-USER` chain which Docker preserves across restarts. Persist rules using `netfilter-persistent save`. Without `ip6tables` enabled, Docker networks have no IPv6 isolation, creating security risks in multi-tenant environments.
