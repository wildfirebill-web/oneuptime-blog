# How to Configure dnsmasq as a DHCP Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, dnsmasq, Linux, DNS, sysadmin, Lightweight

Description: dnsmasq is a lightweight DNS forwarder and DHCP server ideal for home networks, labs, and small deployments, configured through a single file that handles both DNS and DHCP from the same process.

## Installation

```bash
# Debian/Ubuntu
sudo apt install dnsmasq

# RHEL/Fedora
sudo dnf install dnsmasq
```

## Complete Configuration

```
# /etc/dnsmasq.conf

# =============================================
# Interface Configuration
# =============================================
interface=eth0           # Listen only on this interface
bind-interfaces          # Do not listen on other interfaces
no-resolv               # Don't read /etc/resolv.conf for upstream DNS

# Upstream DNS servers
server=8.8.8.8
server=1.1.1.1

# =============================================
# DHCP Configuration
# =============================================
dhcp-range=192.168.1.50,192.168.1.200,255.255.255.0,24h

# DHCP options
dhcp-option=option:router,192.168.1.1
dhcp-option=option:dns-server,192.168.1.1   # Point to dnsmasq itself
dhcp-option=option:domain-name,home.local
dhcp-option=option:ntp-server,129.6.15.28

# =============================================
# Static Assignments (Reservations)
# =============================================
dhcp-host=AA:BB:CC:DD:EE:01,192.168.1.10,server1,infinite
dhcp-host=AA:BB:CC:DD:EE:02,192.168.1.20,printer,infinite

# =============================================
# DNS Local Overrides
# =============================================
address=/server1.home.local/192.168.1.10
address=/printer.home.local/192.168.1.20

# =============================================
# Logging
# =============================================
log-dhcp
log-queries
log-facility=/var/log/dnsmasq.log

# =============================================
# Security
# =============================================
bogus-priv              # Never forward queries for RFC1918 addresses
domain-needed           # Only forward FQDN queries
```

## Start and Test

```bash
# Validate configuration
dnsmasq --test

# Start/enable service
sudo systemctl enable --now dnsmasq

# Follow DHCP log
tail -f /var/log/dnsmasq.log | grep DHCP

# Test DNS resolution through dnsmasq
dig @192.168.1.1 server1.home.local
```

## Multiple Interfaces / VLANs

```
# dnsmasq.conf for multi-VLAN
interface=eth0.10
interface=eth0.20

dhcp-range=tag:eth0.10,10.0.10.50,10.0.10.200,24h
dhcp-range=tag:eth0.20,10.0.20.50,10.0.20.200,24h

dhcp-option=tag:eth0.10,option:router,10.0.10.1
dhcp-option=tag:eth0.20,option:router,10.0.20.1
```

## dnsmasq vs ISC dhcpd

| Feature | dnsmasq | ISC dhcpd |
|---------|---------|-----------|
| DNS forwarding | Yes (built-in) | No |
| Configuration complexity | Low | Medium |
| Failover support | No | Yes |
| DHCP snooping | No | No (switch feature) |
| Best for | SOHO/labs/small sites | Enterprise |

## Key Takeaways

- dnsmasq combines DHCP and DNS in one lightweight daemon — ideal for small networks and labs.
- `dhcp-host=MAC,IP,hostname,lease` handles reservations in a single concise line.
- Use `log-dhcp` to log all DHCP events to a file for debugging.
- `bogus-priv` and `domain-needed` are important security settings — always include them.
